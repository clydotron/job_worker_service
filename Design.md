---
authors: Brian Green (bgreen007@cox.net)
state: draft
---

## What
Job worker service that provides an API to run arbitrary Linux processes.

## Details

### Security
Use mTLS for gRPC communication.

Required files:
| File | Description |
| --- | --- |
| server_cert.pem | the server's certificate (public key) |
| server_key.pem | the server's private key |
| server_ca_cert.pem | the certificate of the CA that can verify the server's certificate. |
| client_cert.pem | the client's certificate (public key). |
| client_key.pem | the client's private key. |
| client_ca_cert.pem | the certificate of the CA that can verify the client's certificate |

Use `openssl` to generate all of the required certs and keys mentioned above.

User authentication and authorization

Simple authorization:
use gRPC interceptors: (will need for both unary and streaming connections)
* On client: add user token
* On server: check the user token against known, reject connection if unknown

TODO need a way to select the current user: 
Options:
command line argument: `... -u alex`
cli command: `jws login <user_name>`
  user name would be stored in an env file and read in at startup

### Privacy
Users can only interact with jobs that they started.

### UX - CLI
Scenarios
Start a job: caller provides the command and arguments in a single string *** explore options
`jws start -c "ls -l" -u alex`
returns the unique identifier of the job

Stop a job:
`jws stop -j <job_id> -u alex`

returns an error if something went wrong (unknown user, invalid job_id, access denied)


Get the status of a job:
`jws status -j <job_id>`

returns the status of the job: "Pending, Active, Terminated"
  or an error: (uknown user, invalid job_id, access denied)

Get the output of a job:
`jws output -j <job_id>`


Sample usage:
`jws start -c "ping google" -u alex`
returns 100

`jws status -j 100 -u alex`
returns Active

`jws status -j 100 -u betty`
returns Access Denied

`jws output -j 100 -u alex`
returns:
...

from a second terminal:
`jsw stop -j 100 -u alex`
returns success (no error)

in the previous terminal: (`jws output -j 100 -u alex`)
sees:
... <insert output here>


### Proto Specifications
```
service JobWorkerService {
  rpc StartJob(StartJobRequest) returns (StartJobResponse);
  rpc StopJob(StopJobRequest) returns (StopJobResponse);
  rpc GetJobStatus(GetJobStatusRequest) returns (GetJobStatusResponse);
  rpc GetJobOutput(GetJobOutputRequest) returns (stream GetJobOutputResponse);
}

message StartJobRequest {
  string cmd = 1;
}

message StartJobResponse {
  string job_id = 1;
}

message StopJobRequest {
  string job_id = 1;
}

message StopJobResponse {}

message GetJobStatusRequest {
  string job_id = 1;
}

message GetJobStatusResponse {
  string status = 1;
}

message GetJobOutputRequest {
  string job_id = 1;
}

message GetJobOutputResponse {
  string output_msg = 1;
}
```

###  Design Details

#### Client
Uses gRPC to connect to the server, implements the CLI described above.
Lifetime: is alive for the duration of the gRPC call, will exit upon completion.
For never ending output streams (like 'ping google') user can use ctrl-c to exit cleanly.
(ctrl-c can also be used for Start/Stop/GetStatus as well)

#### Server
Must be run using `sudo` (required for cgroup actions)
Will be executed within a docker container running Linux to enable the usage of cgroups
Implements the gRPC server for the JobWorkerService described above.
Supports concurrent connections.
Must be able to handle cancellation of all gRPC calls: start/stop/getStatus/getOutput
Must be able to handle a clean shutdown:
* Stop processing inbound requests
* Stop any running jobs
* Close all streaming connections (GetOutput)

Assumptions:
No limit on the amount of space it can use (memory or disk)
Data is not persistent. Close the server, lose the data.
TODO: for persistent storage, use database for 

Implementation details:

JobTracker:
keeps a record of all jobs (map)
Responsible for managing the jobs:
Handles Start/Stop/GetStatus actions
Enforces that only the owner of a process can access it.
Each job is executed in a separate go function and can be canceled.
A mutex will be used to synchronize access to the status of the job.
Standard output (and Std Err) will be redirected to the output logging service

Logging Service:
Is responsible for storing the output of all running processes
Can stream the output to multiple listeners
Uses channels to receive output events from the running process
Uses channels to stream output events to listeners
Supports adding/removing output event listeners for a given job
When a new output event listener is added, all previous output for that job will be streamed to them.


### Test Plan
Unit test coverage for a subset of the code: