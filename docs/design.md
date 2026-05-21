# The Overall Design of Teleleport Challenge 1
## Summary
It is a prototype job worker service designed based on the requirements at https://github.com/gravitational/careers/blob/main/challenges/systems/challenge-1.md. 
## Job
### Definition
The service provides an API for running arbitrary Linux processes. These processes can be any executable programs available on the machine hosting the service. The API supports specifying command, arguments, and resource controls(cpu, memory, ioread, iowrite).

To support job status queries, each job is assigned a unique ID (JobID) using a UUID to prevent exposing process IDs for security concerns. Resource controls for CPU, memory, and disk I/O are implemented per job using cgroups v2, as requested.

```golang
type JobSpec struct {
    JobID     string
    Command   string
    Args      []string
    CPU       int
    Memory    int
    IORead    int
    IOWrite   int
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

The certificates contain the CN (Common Name) field as the username and the OU (Organizational Unit) field as the role for authorization purposes. The server validates these fields against an access control list (ACL), which supports two roles: user and admin. The user role can create jobs and can stop jobs, get job status, and stream job output, but only for jobs they created. They do not have access to jobs created by others. Admin users have access to stop, get status, and stream output for all jobs

We also added resource control as described above. Resource flags are provided as key-value pairs, for example: --resource cpu=1 --resource memory=30m. The current implementation supports cpu (number of CPU cores), memory (memory.max as the memory limit), ioread (disk read throughput limit), and iowrite (disk write throughput limit).


### Start Job
```
client start \
  --server [ip:port] \
  --cert client.crt \
  --key client.key \
  --ca ca.crt \
  --command /usr/bin/python  \
  --arg -m --arg 8080 \
  --cpu=1 
  --memory=30000000
  --ioread=10000000
  --iowrite=5000000
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
  // Indicates whether the job was successfully stopped.
  bool stopped = 1;
}

message GetStatusRequest {
  // ID of the job whose status is being queried.
  string job_id = 1;
}
message GetStatusResponse {
  // Current state of the job.
  JobStatus status = 1;
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
As shown in the architecture, the gRPC server is configured with ca.crt, server.key, and server.crt to enable mTLS and verify client certificates against a trusted Certificated Authority. This is implemented using an authentication interceptor placed before the job service, in order to satisfy the requirement "Use mTLS authentication and verify client certificates.".

The authentication interceptor also checks whether the active cipher belongs to a strong set of TLS cipher suites.
```
var AllowedCipherSuites = map[uint16]bool{
	tls.TLS_AES_256_GCM_SHA384:                  true, // TLS 1.3
	tls.TLS_CHACHA20_POLY1305_SHA256:            true, // TLS 1.3
	tls.TLS_AES_128_GCM_SHA256:                  true, // TLS 1.3
	tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384: true, // TLS 1.2 (Strict Forward Secrecy)
}
...
...
...
// Extract the active cipher suite ID determined during the handshake
activeCipher := tlsInfo.State.CipherSuite
// Reject if the cipher is not explicitly present in our allowed map
if !AllowedCipherSuites[activeCipher] {
    cipherName := tls.CipherSuiteName(activeCipher)
    return status.Errorf(
        codes.Unauthenticated,
        "cryptographic violation: cipher suite %s (0x%X) fails security baseline",
        cipherName,
        activeCipher,
    )
}
```
We might also validate that certificates follow proper cryptographic practices, such as having Subject Alternative Names (SANs).
```
//Enforce Presence of SANs
if len(cert.DNSNames) == 0 && len(cert.IPAddresses) == 0 {
	return fmt.Errorf("security violation: certificate is missing Subject Alternative Names (SANs)")
}
```
An authz that checks the Organizational Unit (OU) field as the role and uses a simple authorization scheme (user and admin), in accordance with the requirement to use a simple authorization scheme.

## Library
### Resource Control
It is implemented using cgroups v2. When a job starts, its resource limits are applied by creating and configuring a cgroup named after its job ID before process execution. A command is constructed from the job’s command and arguments, and the process is started normally. After the process is spawned, its PID (and any child processes, indirectly via process group management) is added to cgroup.procs.

However, there may still be a very small time window between process creation and cgroup assignment during which resource usage is not yet fully constrained.

```
    pid := cmd.Process.Pid
    // 1. Pause the process immediately so it can't consume anything yet
    _ = cmd.Process.Signal(syscall.SIGSTOP)
    // 2. Attach the process to  job cgroup
    procsFile := filepath.Join(cgroupPath, "cgroup.procs")
	if err := os.WriteFile(procsFile, []byte(strconv.Itoa(pid)), 0644); err != nil {
		fmt.Printf("[-] Failed to move PID to cgroup.procs: %v\n", err)
		return
	}
    // 3. Safely resume the process under the new strict boundaries
    _ = cmd.Process.Signal(syscall.SIGCONT)
```
An alternative is to use systemd to run the command, allowing systemd to create and manage the underlying cgroup automatically using its unit and slice configuration, like "/sys/fs/cgroup/jobs/jobs_id"
```
systemd-run --scope --slice=jobs --unit=job_id -p CPUQuota=20% -p MemoryMax=500M command
```

When a stop job request is received, the worker should terminate all processes within the job’s cgroup (as listed in cgroup.procs) by triggering a kill using the following codes. After all processes have exited and the cgroup is empty, the corresponding cgroup directory can be removed. This ensures that all child processes belonging to the job are also terminated, satisfying the requirement that stopping a job must clean up its entire process tree. It can be applied to systemd-run jobs as well.
```
    killFilePath := filepath.Join(cgroupPath, "cgroup.kill")

	// Open the file with write-only permissions.
	// O_WRONLY is required because cgroup.kill is a write-only interface.
	file, err := os.OpenFile(killFilePath, os.O_WRONLY, 0)
	if err != nil {
		return fmt.Errorf("failed to open cgroup.kill: %w", err)
	}
	defer file.Close()

	// Write "1" to trigger the kernel-level SIGKILL cascade.
	_, err = file.WriteString("1")
	if err != nil {
		return fmt.Errorf("failed to write to cgroup.kill: %w", err)
	}
```

### Metadata
A job store is created with an in-memory map where the key is the job id and the value is the job’s metadata. This is used for querying job status to satisify the requirement "Worker library with methods to query status of a job". In addition, each job record is persisted to disk as a JSON file for crash recovery or testing, which might be optional.
 ```
type JobRecord struct {
	JobID      string    `json:"job_id"`
	Command    string    `json:"command"`
	Args       []string  `json:"args"`
    Resources  []string  `json:"resources"`
	Status     string    `json:"status"`
    UserName   string    `json:"user_name"`
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
For streaming output, a running job may have one writer and multiple readers. To efficiently notify multiple readers when new data is written without busy-waiting or polling, use a sync.Cond (Condition Variable) combined with a sync.RWMutex.This approach allows readers to safely suspend execution and sleep until the writer explicitly signals that new data is available, maximizing CPU efficiency to satisfy the requirement "Discovering new output should be efficient, avoid busy-waiting or polling". The sync.RWMutex is also introduced to support multiple concurrent clients. 

Output is persisted on disk, especially for completed jobs, so that readers can start from beginning as "Output should be from start of process execution." required.

When a job completes, times out, fails, or is stopped, a done flag is used to indicate that no more data will arrive, allowing readers to exit cleanly.
```
type SharedFile struct {
	mu      sync.RWMutex
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
Readers and writers operate on raw bytes, without embedding assumptions about the process's output - it may be text or raw binary data.
```
r := bufio.NewReader(outputfile)
buf := make([]byte, 32*1024) 
...
n, err := r.Read(buf)
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