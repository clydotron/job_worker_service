# job_worker_service


## install protoc
## Build Proto files


## Install 


## Generate your own Certs and Keys for mTLS

1. Generate a private key for the CA or cert.
2. Create a Certificate Signing Request (CSR) using the private key
3. Generate the self-signed CA or cert using the CSR and the private key.

Generate

openssl req -new -newkey rsa:2048 -keyout ca.key -x509 -sha256 -days 365 -out ca.crt


(what about if running on Linux)?

