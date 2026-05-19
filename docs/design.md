# The Overall Design of Teleleport Challenge 1
## Summary
It is a prototype job worker service designed based on the requirements at https://github.com/gravitational/careers/blob/main/challenges/systems/challenge-1.md. 
## Job
### Definition
The service provides an API for running arbitrary Linux processes. These processes can be any executable programs available on the machine hosting the service. The API supports specifying commands, arguments, and environment variables.

To support job status queries, each job is assigned a unique ID (UID). Resource controls for CPU, memory, and disk I/O are implemented per job using cgroups, as requested.

There are also some implicit requirements, such as maintaining job status information for querying and supporting role-based access control for authorization (using usernames as a simple approach). Jobs may also have a default timeout value to prevent them from running indefinitely.
```golang
type JobSpec struct {
    UID       string
    Command   []string
    Args      []string
    Env       map[string]string
    Resource  ResourceControl
    Status    JobStatus
    Timeout   time.Duration
    Username  string
    Created   timestamp
    Updated   timestamp
}
```
### Lifecycle
The following diagram illustrates the details of job status. If a job does not exist, unknown is returned by default.

 ![My Local Image](img/job_lifecycle.png)
## Component Architecture
There are three major components: the client, the gRPC server, and the library, as shown below.
 ![My Local Image](img/architecture.png)

## Client
Cobra is integrated to create a root command to support four subcommands: startJobCmd, stopJobCmd, queryStatusCmd, and streamOutputCmd. Since we use mTLS authentication, we verify client certificates. The certificates are required to set up client commands.

We also add resource control as mentioned above. We could use a JSON format in the command line, such as {"cpu": 1, "mem": "30M"}, to make it easier to extend. However, to keep it simple and easy to implement, we use flags like --cpu 1 --mem 30M instead.

We also use .max to define a hard limit for simplicity.

### Start Job
```
client \
  --server [ip:port] \
  --cert client.crt \
  --key client.key \
  --ca ca.crt \
  start <--command> [args...] [envs...] [--cpu] [--mem] [--io] [--timeout]
 ```
### Stop Job
 ```
client \
  --server [ip:port] \
  --cert client.crt \
  --key client.key \
  --ca ca.crt \
  stop <jobUID> [--timeout]
 ```

 ### Query Status
 ```
client \
  --server [ip:port] \
  --cert client.crt \
  --key client.key \
  --ca ca.crt \
  status <jobUID> [--timeout]
 ```

  ### Stream output
 ```
client \
  --server [ip:port] \
  --cert client.crt \
  --key client.key \
  --ca ca.crt \
  output <jobUID> [--timeout]
 ```

   ### (Optional) List jobs
 ```
client \
  --server [ip:port] \
  --cert client.crt \
  --key client.key \
  --ca ca.crt \
  list [--timeout]
 ```

## gPRC Server
Following the commands above, the service proto is defined below. ListJobs is added for testing purposes.
 ```
service JobService {

  // A job is created and return a job UID
  rpc StartJob(StartJobRequest) returns (StartJobResponse);

  // Return whether the job is stopped or not
  rpc StopJob(StopJobRequest) returns (StopJobResponse);

  rpc GetStatus(GetStatusRequest) returns (GetStatusResponse);

  // Streaming output (non-blocking)
  rpc StreamOutput(StreamOutputRequest) returns (stream StreamOutputResponse);

  // Optional for testing
  rpc ListJobs(ListJobsRequest) returns (ListJobsResponse);
}
 ```
As shown in the architecture, an authentication interceptor and an authorization interceptor are placed before the job service to ensure security. We also assume that certificates need to be rotated over time. For simplicity, we currently hardcode the expiration time.

## Library
The job manager provides an interface for the job worker service to start and stop jobs. It includes a resource manager with an upper limit on capacity. When a job is scheduled, the resource manager checks available resources. If insufficient, the job is queued until resources are freed. A FIFO strategy is used for simplicity.

All jobs are persisted as jobUID.json for metadata and jobUID.output for output, including stdout and stderr. A persistenceManager is responsible for managing job metadata and loading all JSON metadata into an in-memory JobStore. A mutex is used to handle concurrent requests. Since output files can be large, a buffer pool is introduced to read files in chunks and stream them to clients accordingly.

## Testing
### Job Lifecycle
Start a job → get job status → stop the job → stream its output -> list all the jobs
Start a job → stop the job → get job status → stream its output -> list all the jobs
Start a job → stream its output until completion -> list all the jobs
Start multiple jobs → stream their outputs -> list all the jobs
Start multiple jobs → stop a job randomly → get job status → stream their outputs -> list all the jobs
Start multiple jobs → stream their outputs → stop a job randomly -> list all the jobs
### Authenication
Access gRPC server without certificates
Access gRPC server with invalid certificates
Access gRPC server with valid certificates
### Authorization
Access own jobs
Access other jobs using user role
Access other jobs using admin role
### Resource controls
Start a job when insufficient resources are available
Queue jobs → release resources → check if new jobs are running
Run jobs → check for resource leaks 