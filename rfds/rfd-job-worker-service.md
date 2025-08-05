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
- Real-time output streaming (tail -f behavior)
- Efficient output discovery mechanisms (stream from any offset)
- Multi-client streaming support with live broadcast
- Non-blocking operations with simultaneous memory storage and client delivery

#### Phase 4: Security & Polish
- User isolation enforcement
- ChaCha20-Poly1305 TLS cipher implementation
- Error handling refinement
- Performance optimization

### Key Design Decisions

**Authentication**: mTLS-only approach eliminates password management complexity while providing strong mutual authentication.

**Output Streaming**: Non-blocking design prevents clients from affecting job execution. Complete process output is stored in memory from the beginning, allowing any client to stream from offset 0 to get the full history. After exhausting stored output, streams remain open and new data is simultaneously saved to memory and broadcast to all connected clients in real-time.

**User Isolation**: Per-user process isolation ensures users can only access jobs they created, with server-side enforcement at the API layer.

**Concurrency**: Standard library synchronization primitives (mutex, channels) to avoid custom threading solutions.


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
  string job_id = 1;
  JobState state = 2;
  int32 exit_code = 3;
  int64 start_time = 4;
  int64 end_time = 5;
  string owner = 6;
}

message StreamOutputRequest {
  string job_id = 1;
  bool follow = 2;
  int64 from_offset = 3;
}

message OutputChunk {
  string job_id = 1;
  bytes data = 2;
  int64 offset = 3;
  OutputType type = 4;
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

# Stream from specific offset (resume/replay)
jobworker stream abc123 --from-offset 1000
# Shows output starting from byte 1000
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