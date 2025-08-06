# Job Worker Service - System Workflows

This document provides detailed workflow diagrams for the job worker service system interactions.

## Job Execution Flow

```
Admin Client                 API Server                    Job Library
     │                           │                             │
     │ StartJob(cmd, args)       │                             │
     ├─────────────────────────▶ │                             │
     │                           │ ValidateAuth(admin)         │
     │                           ├─────────────────────────────│
     │                           │                             │
     │                           │ CreateJob(cmd, args)        │
     │                           ├─────────────────────────────▶
     │                           │                             │
     │                           │ job_id                      │
     │                           │◀─────────────────────────────
     │ JobResponse(job_id)       │                             │
     │◀───────────────────────── │                             │
     │                           │                             │
                                 │                             │
                           [Process Running]                   │
                                 │                             │
     │ StreamOutput(job_id)      │                             │
     ├─────────────────────────▶ │                             │
     │                           │ ValidateAuth(read/admin)    │
     │                           ├─────────────────────────────│
     │                           │                             │
     │                           │ GetOutput(job_id)           │
     │                           ├─────────────────────────────▶
     │                           │                             │
     │                           │ OutputChunk{data}           │
     │                           │◀─────────────────────────────
     │ OutputChunk{data}         │                             │
     │◀───────────────────────── │                             │
     │        ...                │             ...             │
```

## Multi-Client Streaming (Duplicate Windows)

```
Terminal Window 1            Single API Server           Terminal Window 2
(CLI Client A)                                           (CLI Client B)
     │                           │                           │
     │ $ jobctl stream job-123   │                           │
     ├─────────────────────────▶ │                           │
     │                           │                           │
     │                           │   $ jobctl stream job-123 │
     │                           │◀─────────────────────────── │
     │                           │                           │
     │   [Job produces: "Starting process..."]               │
     │                           │                           │
     │ "Starting process..."     │ "Starting process..."     │
     │◀───────────────────────── │─────────────────────────▶ │
     │                           │                           │
     │   [Job produces: "Process completed"]                 │
     │                           │                           │
     │ "Process completed"       │ "Process completed"       │
     │◀───────────────────────── │─────────────────────────▶ │
     │                           │                           │
     │ [BOTH WINDOWS SHOW IDENTICAL OUTPUT IN REAL-TIME]    │
     │                           │                           │

Late-joining scenario (streaming from beginning):
     │                           │                           │
     │                           │   $ jobctl stream job-123 │
     │                           │◀─────────────────────────── │
     │                           │                           │
     │                           │ GetOutput(job_id)         │
     │                           │ [Retrieves ALL stored     │
     │                           │  output from memory]      │
     │                           │─────────────────────────▶ │
     │                           │                           │
     │                           │ "Starting process..."     │
     │                           │ "Initializing..."         │
     │                           │ "Processing data..."       │
     │                           │ "Process completed"       │
     │                           │ [COMPLETE HISTORY FROM    │
     │                           │  PROCESS START]           │
     │                           │◀─────────────────────────── │
     │                           │                           │
     │   [Stream stays open for live updates]                │
     │                           │                           │
     │   [Process produces new output: "Additional data"]    │
     │                           │                           │
     │ "Additional data"         │ 1. Save to memory         │ "Additional data"
     │◀───────────────────────── │ 2. Broadcast to clients  │◀─────────────────────────── │
     │                           │ [SIMULTANEOUS OPERATION]  │                           │
```

## Authentication & Authorization Flow

```
Client Certificate           API Server                 Authorization
     │                          │                           │
     │ mTLS Handshake           │                           │
     ├────────────────────────▶ │                           │
     │                          │ ValidateCert()            │
     │                          ├─────────────────────────▶ │
     │                          │                           │
     │                          │ ExtractUserRole()         │
     │                          ├─────────────────────────▶ │
     │                          │                           │
     │                          │ Role: admin/read-only     │
     │                          │◀───────────────────────── │
     │ Connection Established   │                           │
     │◀──────────────────────── │                           │
     │                          │                           │
     │ API Call                 │                           │
     ├────────────────────────▶ │                           │
     │                          │ CheckPermission(role, op) │
     │                          ├─────────────────────────▶ │
     │                          │                           │
     │                          │ Allow/Deny                │
     │                          │◀───────────────────────── │
     │ Response/Error           │                           │
     │◀──────────────────────── │                           │
```