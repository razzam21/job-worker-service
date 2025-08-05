# RFD: Job Worker Service

**Authors:** Eric  
**State:** Draft

## What

A Go-based job worker service that manages arbitrary Linux processes with secure remote access, real-time output streaming, and progressive feature complexity. The system consists of a reusable library, API server, and CLI client, designed for Level 4 implementation focusing on efficient output discovery and streaming capabilities.

## Why

Current process management solutions lack:
- Secure remote job execution with mTLS authentication
- Real-time output streaming with efficient discovery mechanisms
- Clean separation between job management library and API interfaces
- Progressive complexity that can scale from basic operations to advanced resource control

This service addresses the need for a lightweight, secure, and extensible job management system with per-user process isolation while maintaining simplicity and reliability.

## Details

### Architecture Overview

The system follows a three-tier architecture:

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  CLI Client │───▶│ API Server  │───▶│   Library   │
└─────────────┘    └─────────────┘    └─────────────┘
      │                    │                    │
      │                    │                    │
   mTLS Auth        gRPC/Protobuf      Process Mgmt
```

### System Workflow

Detailed system interaction workflows are documented in [SYSTEM-WORKFLOWS.md](../docs/SYSTEM-WORKFLOWS.md), including:
- Job execution flow with authentication and streaming
- Multi-client streaming with duplicate terminal windows
- Authentication & authorization flow with mTLS

### Core Components

#### 1. Job Management Library
- **Process Lifecycle**: Start, stop, status tracking
- **Output Storage**: Complete process output stored in memory from start to finish
- **Output Streaming**: Non-blocking, tail-f like functionality with full history access and live streaming
- **State Management**: Thread-safe job state with notification mechanisms
- **Resource Tracking**: Process metadata and resource utilization
- **User Ownership**: Track job ownership per user for access control

#### 2. API Server
- **Authentication**: mTLS-only authentication (no custom crypto)
- **Authorization**: Per-user job isolation (users only access their own jobs)
- **Streaming**: Efficient output discovery with multi-client support
- **Concurrency**: Safe handling of multiple simultaneous connections

#### 3. CLI Client
- **Commands**: job start/stop/status/stream operations
- **TLS**: Client certificate authentication
- **Streaming**: Real-time output display with reconnection handling

### Implementation Strategy

#### Phase 1: Foundation (TDD Approach)
- Core library with basic process management
- Simple in-memory job storage
- Basic error handling and logging
- Unit tests for critical paths

#### Phase 2: API Layer
- gRPC server with protobuf definitions
- mTLS authentication implementation
- Certificate generation script (15-day expiration)
- Basic CRUD operations for jobs
- Integration tests

#### Phase 3: Streaming & Discovery
- **Event-Driven Output Capture**: Raw byte-by-byte process output capture using `Read()` blocking (no polling)
- **Atomic Buffer Management**: Thread-safe output storage using `atomic.Value` for race-free concurrent access
- **Notification-Based Discovery**: Channel-based notification system for instant client wake-up on new data
- **Moving Pointer Streaming**: Per-client offset tracking for gap-free delivery from process start
- **Multiple Exit Handling**: Clean termination for client disconnect (Ctrl+C), process completion, and job termination
- **Binary Data Support**: Raw `[]byte` handling without text encoding assumptions for any output type
- **Unified Streaming Model**: Moving pointer seamlessly delivers historical and live data in single loop
- **Lock-Free Architecture**: Channel-only coordination to avoid mutex deadlocks and complexity

**Detailed Implementation**: See [STREAMING-IMPLEMENTATION.md](../docs/STREAMING-IMPLEMENTATION.md) for complete technical specifications, code examples, and performance characteristics.

#### Phase 4: Security & Polish
- User isolation enforcement
- ChaCha20-Poly1305 TLS cipher implementation
- Error handling refinement
- Performance optimization

### Key Design Decisions

**Authentication**: mTLS-only approach eliminates password management complexity while providing strong mutual authentication.

**Output Streaming**: Event-driven capture with atomic buffer storage enables lock-free concurrent access. Moving pointer approach delivers complete history plus live data in unified streaming model. Each client maintains independent offset tracking for gap-free delivery without polling or busy-waiting.

**User Isolation**: Per-user process isolation ensures users can only access jobs they created, with server-side enforcement at the API layer.

**Concurrency**: Channel-based coordination with atomic operations avoids mutex complexity. Lock-free architecture prevents deadlocks while supporting unlimited concurrent streaming clients.


## Security

### Authentication & Authorization
- **mTLS**: Mutual TLS for all client-server communication

### TLS Configuration
- **Cipher Selection**: ChaCha20-Poly1305 chosen over AES-GCM for timing-attack immunity and security robustness
- **Performance Trade-off**: While AES-GCM offers 10-20% better performance on modern hardware with AES-NI, cipher overhead (microseconds) is negligible compared to process spawn times (100ms+) and network latency (10ms+)
- **Security Priority**: Decision prioritizes security over premature optimization, eliminating entire classes of implementation vulnerabilities
- **Certificate Management**: 15-day expiration for demo purposes, 4096-bit RSA keys with SHA-256 signatures, server CN=localhost, client CNs for user identification

### Process Security
- **Command Execution**: Separate command and args fields prevent shell injection attacks using direct process execution (`exec.Command(command, args...)`) instead of shell parsing
- **Process Isolation**: Users cannot access other users' jobs, job ownership verified on all operations
- **Runtime Environment**: Jobs run with server process privileges in controlled environment
- **Resource Protection**: Output streaming limited to prevent resource exhaustion, no arbitrary code execution in server process

## Go Library API Specification

The core library provides a clean interface that the gRPC server uses for job management. This design separates business logic from transport concerns.

### Core Interface

```go
// JobManager provides the main library interface
type JobManager interface {
    StartJob(ctx context.Context, cmd string, args []string, owner string) (string, error)
    StopJob(ctx context.Context, jobID string, owner string) error
    GetJobStatus(ctx context.Context, jobID string, owner string) (*JobStatus, error)
    StreamOutput(ctx context.Context, jobID string, owner string) (<-chan OutputChunk, error)
}

// JobStatus represents current job state
type JobStatus struct {
    JobID     string
    State     JobState
    ExitCode  int32
    StartTime time.Time
    EndTime   time.Time
    Owner     string
}

// OutputChunk represents streaming output data
type OutputChunk struct {
    Data   []byte
    Offset int64
    Type   OutputType
}

// JobState represents job execution state
type JobState int32

const (
    JobStateUnknown JobState = iota
    JobStateRunning
    JobStateCompleted
    JobStateFailed
    JobStateStopped
)

// OutputType distinguishes stdout/stderr
type OutputType int32

const (
    OutputTypeStdout OutputType = iota
    OutputTypeStderr
)
```

### Implementation Structure

```go
// jobManager implements JobManager interface
type jobManager struct {
    jobs   map[string]*Job
    jobsMu sync.RWMutex
}

// Job represents a running process with streaming capabilities
type Job struct {
    jobID     string
    process   *exec.Cmd
    outputCh  chan struct{}
    doneCh    chan struct{}
    buffer    atomic.Value  // []byte
    owner     string
    state     atomic.Value  // JobState
    startTime time.Time
    endTime   atomic.Value  // time.Time
}
```

### Usage Examples

```go
// Initialize job manager
manager := NewJobManager()

// Start a job
jobID, err := manager.StartJob(ctx, "ping", []string{"google.com"}, "user1")
if err != nil {
    return err
}

// Stream output
outputCh, err := manager.StreamOutput(ctx, jobID, "user1")
if err != nil {
    return err
}

// Process streaming data
for chunk := range outputCh {
    fmt.Printf("Received %d bytes at offset %d\n", len(chunk.Data), chunk.Offset)
    os.Stdout.Write(chunk.Data)
}

// Get job status
status, err := manager.GetJobStatus(ctx, jobID, "user1")
if err != nil {
    return err
}
fmt.Printf("Job %s is %v\n", status.JobID, status.State)

// Stop job
err = manager.StopJob(ctx, jobID, "user1")
```

### Error Handling

```go
var (
    ErrJobNotFound      = errors.New("job not found")
    ErrPermissionDenied = errors.New("permission denied")
    ErrJobAlreadyDone   = errors.New("job already completed")
    ErrInvalidCommand   = errors.New("invalid command")
)
```

### Key Design Principles

- **Owner-based Authorization**: All operations require owner parameter for user isolation
- **Context Propagation**: All methods accept context for cancellation and timeouts
- **Streaming Channels**: Output streaming returns Go channels for natural concurrency
- **Atomic State**: Thread-safe state management using atomic operations
- **Error Types**: Specific error types enable proper gRPC status code mapping

## Proto Specification

```protobuf
syntax = "proto3";
package jobworker;

service JobWorker {
  rpc StartJob(StartJobRequest) returns (StartJobResponse);
  rpc StopJob(StopJobRequest) returns (StopJobResponse);
  rpc GetJobStatus(GetJobStatusRequest) returns (GetJobStatusResponse);
  rpc StreamOutput(StreamOutputRequest) returns (stream OutputChunk);
}

message StartJobRequest {
  string command = 1;
  repeated string args = 2;
}

message StartJobResponse {
  string job_id = 1;
  string owner = 2;
}

message StopJobRequest {
  string job_id = 1;
}

message StopJobResponse {
  bool success = 1;
}

message GetJobStatusRequest {
  string job_id = 1;
}

message GetJobStatusResponse {
  JobState state = 1;
  int32 exit_code = 2;
  int64 start_time = 3;
  int64 end_time = 4;
  string owner = 5;
}

message StreamOutputRequest {
  string job_id = 1;
}

message OutputChunk {
  bytes data = 1;
  int64 offset = 2;
  OutputType type = 3;
}

enum JobState {
  UNKNOWN = 0;
  RUNNING = 1;
  COMPLETED = 2;
  FAILED = 3;
  STOPPED = 4;
}

enum OutputType {
  STDOUT = 0;
  STDERR = 1;
}
```

## CLI Usage Examples

```bash
# Start a job (as user1)
jobworker start ping google.com
# Output: Job started with ID: abc123

# Stream job output in real-time
jobworker stream abc123
# Shows live output from the ping command

# Check job status
jobworker status abc123
# Output: Job abc123 - RUNNING - Started: 2023-12-01 10:30:15

# Stop a running job
jobworker stop abc123
# Output: Job abc123 stopped successfully

# Multiple clients can stream simultaneously
jobworker stream abc123  # In second terminal
# Each client gets complete history + live output
```

## Test Plan

### Unit Tests
- Job lifecycle management
- Output streaming mechanics
- User authorization logic
- Concurrent access safety

### Integration Tests  
- End-to-end job execution
- mTLS authentication flows
- Multi-client streaming scenarios
- Error condition handling

### Manual Testing
- CLI usability validation
- Performance under load
- Network failure recovery
- Security boundary verification

## Implementation Notes

**Hardcoded Values Preference**: Initial implementation deliberately uses hardcoded values for quick setup and testing:
- Server port: 8443 (hardcoded)
- Certificate paths: "./certs/server.crt", "./certs/client1.crt", "./certs/client2.crt", etc.
- Timeouts: 30s connection, 5s read/write
- Buffer sizes: 4KB for output streaming
- Job storage: In-memory map with mutex
- Output storage: Complete process output retained in memory per job
- User identity: Extracted from certificate CN field for job ownership tracking

This approach prioritizes getting the system running quickly over configuration flexibility. Production deployment can add configuration layers later with TODO comments marking these areas.

**Error Handling**: Comprehensive error wrapping with context, avoiding silent failures.

**Logging**: Structured logging for operational visibility without exposing sensitive data.

**Dependencies**: Minimal external dependencies documented in [DEPENDENCIES.md](../docs/DEPENDENCIES.md), preferring standard library solutions.