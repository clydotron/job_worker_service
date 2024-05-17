---
authors: Brian Green (bgreen007@cox.net)
state: draft
---

## What
Job worker service that provides an API to run arbitrary Linux processes.

## Details


### Security
Use mTLS for gRPC communication. TLS version 1.3 only.

Create a self-signed root certificate authority (CA) to sign the certificates used by the clients and server. This CA will also be used to verify the authenticity of the certs during gRPC authentication. Using RSA 2048.

Required files: Server
| File | Description |
| --- | --- |
| server_cert.pem | the server's certificate (public key) |
| server_key.pem | the server's private key |
| root_ca.pem | the certificate of the CA that can verify the client's certificate |

Required files: Client:
| File | Description |
| --- | --- |
| client_cert.pem | the client's certificate (public key). |
| client_key.pem | the client's private key. |
| root_ca.pem | the certificate of the CA that can verify the server's certificate. |

Use CloudFlare's `cfssl` CLI to create and sign all of the certs mentioned above.

#### Creating the self-signed root CA
`cfssl selfsign -config cfssl.json --profile rootca "Greenbeans Dev CA" csr.json | cfssljson -bare root`

[cfssl.json](cfssl.json)
[csr.json](csr.json)

# Creating and signing the server certificate:
```
cfssl genkey csr.json | cfssljson -bare server
cfssl sign -ca root.pem -ca-key root-key.pem -config cfssl.json -profile server server.csr | cfssljson -bare server

```

# Creating and signing the client(s) certificates:
```
cfssl genkey csr.json | cfssljson -bare client
cfssl sign -ca root.pem -ca-key root-key.pem -config cfssl.json -profile client client.csr | cfssljson -bare client
```
Multiple clients are supported, but each client will require it own certificate. The calls above can easily be modified to include additional client information (name):
```
cfssl genkey csr.json | cfssljson -bare client_alex
cfssl sign -ca root.pem -ca-key root-key.pem -config cfssl.json -profile client client_alex.csr | cfssljson -bare client_alex
```
The CLI will include an optional parameter to specify the `client` key name, otherwise will default to 'client'.


### Privacy
Users can only interact with jobs that they started.

### UX - CLI
Scenarios<br />
Start a job:<br />
caller provides the command and arguments in a single string *** explore options 
`jws start -c "ls -l"`<br />
returns the unique identifier of the job 

Stop a job:<br />
`jws stop -j <job_id>`<br />
returns an error if something went wrong (unknown user, invalid job_id, access denied)

Get the status of a job:<br />
`jws status -j <job_id>`<br />
returns the status of the job: "Pending, Active, Terminated"
  or an error: (uknown user, invalid job_id, access denied)

Get the output of a job:<br />
`jws output -j <job_id>`<br />


### Proto Specifications
```
service JobWorkerService {
  // Starts a job using the specified command and arguments, 
  // will return immediatelly after starting the job with the unique id of the job
  rpc StartJob(StartJobRequest) returns (StartJobResponse);

  // Stops the specified job, blocks until the job finishes.
  rpc StopJob(StopJobRequest) returns (google.protobuf.empty);

  // Gets the status of specified job, returns immediately. See enum below for possible values.
  rpc GetJobStatus(GetJobStatusRequest) returns (GetJobStatusResponse);

  // Gets the output of the specified job: uses server streaming to deliver the output.
  // If the specified job does not exist, will return an error and close the stream
  // If the job does exist, returns all stored output for the job.
  // if the job has ended (status is terminated or user_cancelled), will close the stream.
  // if the status is active, will return all additional output as it is generated.
  // The stream will automatically be closed by the server when the job terminates or is stopped by the user.
  // If the client hits ctrl-c, or is terminated, the stream will closed from client side.
  // For simplicty, server does not keep any information related to what client has seen what. 
  // You call this RPC, you get all the output.
  rpc GetJobOutput(GetJobOutputRequest) returns (stream GetJobOutputResponse);
}

// the request message containing the comand (with arguments to start)
message StartJobRequest {
  string cmd = 1;
}

// the response message containing the id of the newly started job
message StartJobResponse {
  string job_id = 1;
}

// the request to stop the job with the given job_id
message StopJobRequest {
  string job_id = 1;
}

// the request to get the status of the specified job
message GetJobStatusRequest {
  string job_id = 1;
}

// the response containing the status of the specified job
message GetJobStatusResponse {
  // an enum containing all possible job statuses
  enum JobStatus {
    JOB_STATUS_UNDEFINED = 0;
    JOB_STATUS_PENDING = 1;
    JOB_STATUS_ACTIVE = 2;
    JOB_STATUS_USER_CANCELLED = 3;
    JOB_STATUS_TERMINATEDS = 4
  }

  JobStatus status = 1;
}

// the request to get the output of a specified job
message GetJobOutputRequest {
  string job_id = 1;
}

// a response containing output from the specified job, can be from either stdout or stderr.
// simplify this: return only 'output', can be either stdout or stderr
message GetJobOutputResponse {
  oneof output_msg {
    bytes stdout = 1;
    bytes stderr = 2;
  }
}
```

###  Design Details

#### Client
Uses gRPC to connect to the server, implements the CLI described above.<br />
Each unique client must have its own certificate 
Lifetime: is alive for the duration of the gRPC call, will exit upon completion.<br />
For never ending output streams (like 'ping google') user can use ctrl-c to exit cleanly.<br />
(ctrl-c can also be used for Start/Stop/GetStatus as well)<br />

#### Server
Must be run using `sudo` (required for cgroup actions)<br />
Can be executed directly on Linux or within a docker container running Linux if on macos or Windows<br />
Implements the gRPC server for the JobWorkerService described above.<br />
Supports concurrent connections from multiple clients<br />
Must be able to handle cancellation of all gRPC calls: start/stop/getStatus/getOutput<br />
Must be able to handle a clean shutdown:
* Stop processing inbound requests
* Stop any running jobs
* Close all streaming connections (GetOutput)

Assumptions:
* No limit on the amount of space it can use (memory or disk)
* Data is not persistent. Close the server, lose the data.

##### Implementation details:

JobTracker:
* keeps a record of all jobs (map) 
* Responsible for managing the jobs: 
* Handles Start/Stop/GetStatus actions 
* Enforces that only the owner of a process can access it. 
* Each job is executed in a separate go function and can be canceled. 
* A mutex will be used to synchronize access to the status of the job. 
* Standard output (Stdout) and StdErr will be redirected to the output logging service
* Uses cgroups v2 to manage the CPU, Memory and Disk IO usage per job.

| Resource | CGroup name | setting(s) |
| --- | --- | --- |
| CPU | cpu | cfs_period, cfs_quota_us |
| Memory | memory | limit_in_bytes |
| Disk IO | io | max |


Logging Service: 
* Is responsible for storing the output of all running processes
* Can stream the output to multiple listeners
* Uses Go channels to receive output events from the running process
* Uses Go channels to stream output events to listeners
* Supports adding/removing output event listeners for a given job
* When a new output event listener is added, all previous output for that job will be streamed to them.



### Test Plan
Unit test coverage for a subset of the code:
* Authentication
* Networking
* Unhappy/error scenario.
