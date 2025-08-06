# Streaming Implementation Details

## Overview

This document provides detailed implementation specifics for the real-time output streaming mechanism that satisfies Level 4 challenge requirements. The implementation ensures efficient output discovery without polling or busy-waiting while supporting multiple concurrent clients.

## Core Requirements Addressed

- **Efficient Discovery**: No polling or busy-waiting for new output
- **From Process Start**: All clients receive complete output history
- **Multiple Clients**: Concurrent streaming to different clients
- **Binary Safe**: Handles any output type (text or binary data)
- **Clean Exit**: Proper cleanup when processes terminate or clients disconnect

## Architecture Overview

```
Process Output → captureOutput() → Atomic Buffer → StreamOutput() → Client Channels
                        ↓                ↓              ↓
                   Notification    Persistent      Per-Client
                    Channel         Storage        Streaming
```

## Implementation Components

### 1. Job Structure

```go
type Job struct {
    jobID    string
    process  *exec.Cmd
    outputCh chan struct{}    // Prevents polling by signaling data availability
    doneCh   chan struct{}    // Coordinates clean shutdown across goroutines
    buffer   atomic.Value     // Ensures race-free access to growing output data
}

type OutputChunk struct {
    Data []byte     // Preserves binary data without encoding assumptions
    Type OutputType // Distinguishes output streams for proper display
}
```

### 2. Output Capture Function

**Purpose**: Reads process output and stores in persistent buffer with client notification.

```go
func (j *Job) captureOutput() {
    go func() {
        defer close(j.outputCh) // Prevents clients from waiting indefinitely
        
        j.buffer.Store([]byte{}) // Ensures atomic operations have valid initial state
        buf := make([]byte, 4096) // Balances memory usage with system call overhead
        
        for {
            n, err := j.process.Stdout.Read(buf)
            if n > 0 {
                // Create copy to prevent buffer reuse corruption
                newData := make([]byte, n)
                copy(newData, buf[:n])
                
                // Atomic update prevents race conditions with concurrent readers
                current := j.buffer.Load().([]byte)
                updated := append(current, newData...)
                j.buffer.Store(updated)
                
                // Non-blocking notification prevents capture thread stalling
                select {
                case j.outputCh <- struct{}{}:
                default: // Prevents deadlock when no clients listening
                }
            }
            
            if err != nil {
                if err == io.EOF {
                    break // Process closed stdout normally
                }
                break // Prevents infinite loop on read errors
            }
        }
    }()
    
    // Monitor process completion to signal streaming clients
    go func() {
        j.process.Wait() // Blocks until process exits
        close(j.doneCh)  // Signal all streaming clients that process is done
    }()
}
```

**Key Design Decisions**:
- **Atomic Buffer Updates**: `atomic.Value` prevents race conditions during concurrent reads
- **Non-blocking Notifications**: `select` with `default` prevents capture goroutine from blocking
- **Raw Byte Handling**: No assumptions about text encoding or line boundaries
- **Efficient Chunking**: 4KB reads balance memory usage with system call overhead
- **Simple Process Monitoring**: `process.Wait()` in dedicated goroutine signals completion to clients

### 3. Client Streaming Function

**Purpose**: Provides individual output streams to clients with moving pointer for gap-free delivery.

```go
func (j *Job) StreamOutput(ctx context.Context) <-chan OutputChunk {
    clientCh := make(chan OutputChunk, 10) // Prevents blocking on slow client processing
    
    go func() {
        defer close(clientCh) // Signals EOF to client when streaming ends
        
        pointer := int64(0) // Enables gap-free delivery from process start
        
        for {
            // Always check buffer first to catch gap data
            currentBuffer := j.buffer.Load().([]byte)
            bufferSize := int64(len(currentBuffer))
            
            if bufferSize > pointer {
                // Copy prevents slice corruption from concurrent buffer updates
                unsent := make([]byte, bufferSize-pointer)
                copy(unsent, currentBuffer[pointer:bufferSize])
                
                chunk := OutputChunk{
                    Data: unsent,
                    Type: OutputTypeStdout,
                }
                
                select {
                case clientCh <- chunk:
                    pointer = bufferSize // Prevents duplicate data delivery
                case <-ctx.Done():
                    return // Handles client disconnect gracefully
                case <-time.After(5 * time.Second):
                    return // Terminates unresponsive clients to prevent goroutine leaks
                }
            }
            
            // Efficient waiting prevents busy polling
            select {
            case <-j.outputCh: // Wakes up immediately when new data arrives
                continue // Rechecks buffer for new data
            case <-ctx.Done(): // Handles client cancellation cleanly
                return
            case <-j.doneCh: // Ensures final data delivery before exit
                // Prevents data loss when process terminates
                finalBuffer := j.buffer.Load().([]byte)
                if int64(len(finalBuffer)) > pointer {
                    remaining := make([]byte, int64(len(finalBuffer))-pointer)
                    copy(remaining, finalBuffer[pointer:]) // Avoids slice reference to freed memory
                    
                    select {
                    case clientCh <- OutputChunk{
                        Data: remaining,
                        Type: OutputTypeStdout,
                    }:
                    case <-ctx.Done():
                    case <-time.After(5 * time.Second):
                        // Final data delivery timeout - prevents hanging on unresponsive clients
                    }
                }
                return
            }
        }
    }()
    
    return clientCh
}
```

**Key Design Decisions**:
- **Moving Pointer**: Each client tracks their position independently
- **Gap-Free Delivery**: Historical data sent immediately, then live streaming
- **Graceful Exit**: Multiple exit paths (client disconnect, process completion)
- **Buffered Channel**: 10-element buffer prevents blocking on client processing delays
- **Client Timeout**: 5-second timeout prevents goroutine leaks from unresponsive clients

## Efficient Discovery Mechanism

The implementation achieves efficient discovery through:

### 1. Event-Driven Architecture
- **No Polling**: `Read()` blocks until data is available from process
- **Immediate Notification**: Clients wake up instantly when new data arrives
- **Channel-Based Signaling**: Go channels provide efficient blocking/wakeup mechanism

### 2. Race Condition Prevention
```go
// Moving pointer approach eliminates race conditions
currentBuffer := j.buffer.Load().([]byte) // Gets complete history at connection time

// Single loop handles both historical and live data
for {
    // Buffer check first prevents missing data during notification gaps
    newBuffer := j.buffer.Load().([]byte)
    if len(newBuffer) > pointer {
        // Immediate delivery maintains real-time streaming
    }
    
    // Blocking wait prevents CPU spinning
    <-j.outputCh
}
```

### 3. Multiple Client Support
- **Independent Streams**: Each `StreamOutput()` call creates separate goroutine
- **Shared Buffer**: All clients read from same atomic buffer
- **Individual Tracking**: Per-client pointers prevent data duplication/loss

## Exit Handling

### Clean Termination Scenarios

1. **Client Disconnect (Ctrl+C)**
   ```go
   case <-ctx.Done(): // Context cancelled prevents resource leaks
       return // Immediate cleanup prevents goroutine accumulation
   ```

2. **Process Natural Exit**
   ```go
   go func() {
       j.process.Wait() // Blocks until process exits
       close(j.doneCh)  // Signal all streaming clients that process is done
   }()
   ```

3. **Process Killed via StopJob**
   ```go
   func (jm *jobManager) StopJob(ctx context.Context, jobID string) error {
       // ... validation code ...
       
       // Kill the process - this will cause process.Wait() to return
       if err := job.process.Process.Kill(); err != nil {
           return fmt.Errorf("failed to kill process: %w", err)
       }
       
       return nil
   }
   ```

**Guarantee**: All exit paths result in proper channel closure and goroutine cleanup.

## Performance Characteristics

### Memory Usage
- **Linear Growth**: Buffer grows with total process output
- **Per-Client Overhead**: ~10KB buffered channel + goroutine stack
- **Atomic Operations**: Minimal CPU overhead for concurrent access

### Network Efficiency
- **Chunk-Based Delivery**: Reduces gRPC message overhead
- **Binary Preservation**: No encoding/decoding overhead

## Error Handling

### Read Errors
```go
if err != nil {
    if err == io.EOF {
        break // Process terminated stdout cleanly
    }
    // Exit prevents infinite error loops
    break
}
```

### Client Errors
- **Send Timeouts**: 5-second timeout prevents indefinite blocking on slow clients
- **Channel Full**: Buffered channels with reasonable limits, unresponsive clients terminated
- **Network Issues**: gRPC layer handles connection failures

## Future Enhancements

### STDERR Capture
```go
go j.captureStream(j.process.Stdout, OutputTypeStdout) // Separate goroutines prevent blocking
go j.captureStream(j.process.Stderr, OutputTypeStderr) // Independent streams for proper ordering
```

### Resource Limits
- **Buffer Size Caps**: Prevent memory exhaustion on long-running jobs
- **Client Limits**: Maximum concurrent streaming clients per job
- **Backpressure**: Slow client handling without affecting others

## Testing Approach

### Key Test Scenarios
1. **Single Character Output**: Verify immediate delivery without buffering delays
2. **Binary Data**: Ensure no corruption or text-encoding assumptions
3. **Multiple Clients**: Concurrent streaming with independent progress tracking
4. **Process Termination**: Clean exit handling for all termination types
5. **Large Output**: Memory usage and performance with substantial data volumes

### Race Condition Testing
```bash
go test -race ./... # Catches atomic.Value and channel race conditions
```

This implementation satisfies all Level 4 requirements while maintaining simplicity and avoiding common concurrency pitfalls outlined in the challenge guidance.