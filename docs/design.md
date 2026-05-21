# The Overall Design of Teleleport Challenge 1
## Summary
It is a prototype job worker service designed based on the requirements at https://github.com/gravitational/careers/blob/main/challenges/systems/challenge-1.md. 
## Job
### Definition
The service provides an API for running arbitrary Linux processes. These processes can be any executable programs available on the machine hosting the service. The API supports specifying commands, arguments, and environment variables.

To support job status queries, each job is assigned a unique ID (JobID) using a UUID to prevent exposing process IDs for security concerns. Resource controls for CPU, memory, and disk I/O are implemented per job using cgroups v2, as requested.

```golang
type JobSpec struct {
    JobID     string
    Command   string
    Args      []string
    Resources map[string]string
    ProcessID int
    Status    JobStatus
    Username  string
}
const (
	Running   JobStatus = iota
	Completed          // process exited 0
	Stopped            // killed via Stop()
	Failed             // process exited non-zero         
    Unknown
)
```
## Component Architecture
There are three major components: the client, the gRPC server, and the library, as shown below. The greyed-out ones are optional.

 ![My Local Image](img/architecture.png)

## Client
Cobra is integrated to create a root command that supports four subcommands: startJobCmd, stopJobCmd, queryStatusCmd, and streamOutputCmd, which satisfies the requirement "CLI should be able to connect to worker service and start, stop, get status, and stream output of a job."

Since we use mTLS authentication, client certificates are presented to the server during the TLS handshake for verification. These certificates are used to authenticate the client and authorize requests.

The certificates contain the CN (Common Name) field as the username and the OU (Organizational Unit) field as the role for authorization purposes. The server validates these fields against an access control list (ACL), which supports two roles: user and admin. The user role can create jobs and can stop jobs, get job status, and stream job output, but only for jobs they created. They do not have access to jobs created by others. Admin users have access to stop , get status, and stream output for all jobs

We also added resource control as described above. Resource flags are provided as key-value pairs, for example: --resource cpu=1 --resource memory=30m. The current implementation supports cpu (number of CPU cores), memory (memory.max as the memory limit), io.read (disk read throughput limit), and io.write (disk write throughput limit).


### Start Job
```
client start \
  --server [ip:port] \
  --cert client.crt \
  --key client.key \
  --ca ca.crt \
  --command /usr/bin/python  \
  --arg -m --arg 8080 \
  --resource cpu=1 --resource memory=30m --resource io.read=10mb --resource io.write=5mb
 ```
### Stop Job
 ```
client stop \
  --server [ip:port] \
  --cert client.crt \
  --key client.key \
  --ca ca.crt \
  --id  cb056f6e-1643-44d3-9f64-11688bc562c4
 ```

 ### Query Status
 ```
client status \
  --server [ip:port] \
  --cert client.crt \
  --key client.key \
  --ca ca.crt \
  --id  cb056f6e-1643-44d3-9f64-11688bc562c4
 ```

  ### Stream output
 ```
client output \
  --server [ip:port] \
  --cert client.crt \
  --key client.key \
  --ca ca.crt \
  --id  cb056f6e-1643-44d3-9f64-11688bc562c4
 ```


## gPRC Server

Following the commands above, the service proto is defined below to satisfy the requirement "gRPC API to start/stop/get status/stream output of a job."
 ```
service JobService {

  // A job is created and return a job ID
  // The job is executed asynchronously.
  rpc StartJob(StartJobRequest) returns (StartJobResponse);

  // Return whether the job is stopped or not
  // Returns whether the stop operation was successfully applied.
  rpc StopJob(StopJobRequest) returns (StopJobResponse);

  // Retrieves the current status of a job (e.g., RUNNING, STOPPED).
  rpc GetStatus(GetStatusRequest) returns (GetStatusResponse);

  // Streaming output (non-blocking) from a running, completed, failed, or stopped job
  rpc StreamOutput(StreamOutputRequest) returns (stream StreamOutputResponse);

}

enum JobStatus {
  // Default value
  JOB_STATUS_UNKNOWN = 0;
  // Job is currently running.
  JOB_STATUS_RUNNING = 1;
  // Job has been stopped by a user.
  JOB_STATUS_STOPPED = 2;
  // Job has failed due to an error.
  JOB_STATUS_FAILED = 3;
  // Job completed successfully.
  JOB_STATUS_COMPLETED = 4;
}

message StartJobRequest {
  // Command to execute
  string command = 1;
  // Arguments passed to the command.
  repeated string args = 2;
  // Resource constraints for the job (e.g., CPU, memory, and io).
  // Represented as key-value pairs.
  map<string, string> resources = 3;
}
message StartJobResponse {
  // Unique identifier assigned to the created job.
  string job_id = 1;
}

message StopJobRequest {
  // ID of the job to stop.
  string job_id = 1;
}
message StopJobResponse {
  // ID of the job that was requested to stop.
  string job_id = 1;
  // Indicates whether the job was successfully stopped.
  bool stopped = 2;
}

message GetStatusRequest {
  // ID of the job whose status is being queried.
  string job_id = 1;
}
message GetStatusResponse {
  // ID of the job being queried.
  string job_id = 1;
  // Current state of the job.
  JobStatus status = 2;
}

message StreamOutputRequest {
  // ID of the job whose output stream is requested.
  string job_id = 1;
}
message StreamOutputResponse {
  // Chunk of raw output data from the job.
  // Typically represents stdout and stderr stream bytes.
  bytes payload = 1; 
}

 ```
As shown in the architecture, the gRPC server is configured with ca.crt, server.key, and server.crt to enable mTLS and verify client certificates. This is implemented using an authentication interceptor placed before the job service, in order to satisfy the requirement: ‘Use mTLS authentication and verify client certificates. Set up a strong set of cipher suites for TLS and a secure cryptographic configuration for certificates. Do not use any other authentication protocols on top of mTLS.

An authorization interceptor that checks the Organizational Unit (OU) field as the role and uses a simple authorization scheme (user and admin), in accordance with the requirement ‘Use a simple authorization scheme.

## Library
### Resource Control
It is implemented using cgroups v2. When a job starts, its resource limits are applied by creating a cgroup named after its job id. A command is constructed from the job’s command and argument attributes. After the command is started, the process ID (PID) and any child process IDs are added to cgroup.procs (use SIGSTOP and resume with SIGCONT to get child processes for race condition concerns).

When a stop job request is received, the worker should terminate all processes within the job’s cgroup (as listed in cgroup.procs) by sending appropriate termination signals. After all processes have exited and the cgroup is empty, the corresponding cgroup directory can be removed. This ensures that all child processes belonging to the job are also terminated, satisfying the requirement that stopping a job must clean up its entire process tree.

### Metadata
A job store is created with an in-memory map where the key is the job id and the value is the job’s metadata. This is used for querying job status to satisify the requirement "Worker library with methods to query status of a job". In addition, each job record is persisted to disk as a JSON file for crash recovery or testing, which might be optional.
 ```
type JobRecord struct {
	JobID      string    `json:"job_id"`
	Command    string    `json:"command"`
	Args       []string  `json:"args"`
    Resources  []string  `json:"resources"`
	Status     string    `json:"status"`
	PID        int       `json:"pid"`
	ExitCode   int       `json:"exit_code"`
	StartTime  time.Time `json:"start_time"`
	EndTime    time.Time `json:"end_time,omitempty"`
	Error      string    `json:"error,omitempty"`
	OutputPath string    `json:"output_path"` 
}
type JobStore struct {
	mu      sync.RWMutex //For reading and writing the job record map
	records map[string]*JobRecord
}
 ```
### Output
For streaming output, a running job may have one writer and multiple readers. To efficiently notify multiple readers when new data is written without busy-waiting or polling, use a sync.Cond (Condition Variable) combined with a sync.RWMutex.This approach allows readers to safely suspend execution and sleep until the writer explicitly signals that new data is available, maximizing CPU efficiency to satisfy the requirement "Discovering new output should be efficient, avoid busy-waiting or polling". The sync.RWMutex is also introduced to support multiple concurrent clients. Readers and writers operate on raw byte slices, without embedding assumptions about the process's output - it may be text or raw binary data.

Output is persisted on disk, especially for completed jobs, so that readers can start from beginning as "Output should be from start of process execution." required.

When a job completes, times out, fails, or is stopped, a done flag is used to indicate that no more data will arrive, allowing readers to exit cleanly.
```
type SharedFile struct {
	mu      sync.Mutex
	cond    *sync.Cond
	version uint64
	done    bool
}
 ```

Writer 
```
sharedFile.mu.Lock()
defer sharedFile.mu.Unlock()
...
...
...

sharedFile.version++        // Update state
sharedFile.cond.Broadcast() // Wake up all waiting readers efficiently
```

Reader
```
sharedFile.mu.RLock()
defer sharedFile.mu.RUnlock()
...
...
...

for sharedFile.version == last && !sharedFile.done{
	sharedFile.cond.Wait()
}

```


## Testing
### Job Lifecycle
Start a job → get job status → stop the job → stream its output → list all the jobs

Start a job → stop the job → get job status → stream its output → list all the jobs

Start a job → stop the job → get job status → stop the job again → list all the jobs

Start a job → stream its output until completion → list all the jobs

Start a long-run job to hit timeout limit → stream its output → get job status → list all the jobs

Start a job with errors → stream its output → get job status → list all the jobs

Start multiple jobs → stream their outputs → list all the jobs

Start multiple jobs → stop jobs randomly → get job status → stream their outputs → list all the jobs

Start multiple jobs → stream their outputs → stop jobs randomly → list all the jobs

Start different type jobs(normal, timeout, error) jobs → stop jobs randomly → get job status → stream their outputs → list all the jobs

Start jobs → use mutiple clients to stream their outputs when the job are running

Start jobs → use mutiple clients to stream their outputs after the job are completed

### Authenication
Access gRPC server without certificates

Access gRPC server with invalid certificates

Access gRPC server with valid certificates
### Authorization
Access own jobs

Access other jobs using user role

Access other jobs using admin role

### Resource controls

Pass large resource limits that the current system fails to meet → check if new jobs are running

Pass invalid resource limits → check if new jobs are running

Pass valid resource limits → check if new jobs are running

Run jobs → check for resource leaks 