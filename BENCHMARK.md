# Performance Benchmark: rmate Implementations

Comparison of four `rmate` implementations: C, Bash, Python, and Ruby.

## Test Environment

- **OS**: Linux x86\_64 (RHEL 8, kernel 4.18)
- **CPU**: shared server
- **Bash**: 4.4.20
- **Python**: 3.13.3
- **Ruby**: 2.5.9
- **GCC**: system default, `-O2`
- **Method**: wall-clock time via `time` built-in; memory via `/usr/bin/time -v`
  (Maximum resident set size). A dummy TCP server accepted connections, consumed
  all data, and replied with a `close` command so each client ran its full
  lifecycle including protocol send and receive.

## 1. Startup Time

100 iterations of `--version` (pure startup + exit, no I/O).

| Implementation | Total (100 runs) | Per-invocation |
|----------------|------------------:|---------------:|
| **C**          |        **0.033s** |    **0.3 ms**  |
| Bash           |           0.218s  |       2.2 ms   |
| Python         |           3.405s  |      34.0 ms   |
| Ruby           |           4.346s  |      43.5 ms   |

C is ~7x faster than Bash and ~100–130x faster than the scripted versions.
The scripting languages pay a fixed interpreter startup cost on every
invocation.

## 2. End-to-End File Send (wait mode)

Each run opens a TCP connection, sends the protocol header + file data, reads
the server's `close` reply, and exits.

### 1 KB file (100 iterations)

| Implementation | Total   |
|----------------|--------:|
| **C**          | **0.052s** |
| Bash           |   0.563s |
| Python         |   3.698s |
| Ruby           |   4.397s |

### 1 MB file (20 iterations)

| Implementation | Total   |
|----------------|--------:|
| **C**          | **0.118s** |
| Bash           |   0.140s |
| Python         |   0.779s |
| Ruby           |   0.918s |

### 10 MB file (5 iterations)

| Implementation | Total   |
|----------------|--------:|
| **C**          |   0.379s |
| **Bash**       | **0.329s** |
| Python         |   0.498s |
| Ruby           |   0.527s |

### 100 MB file (1 iteration)

| Implementation | Total   |
|----------------|--------:|
| C              |   9.58s |
| **Bash**       | **8.44s** |
| Python         |   8.51s |
| Ruby           |   8.47s |

**Analysis**: For small files, startup cost dominates — C wins by a wide
margin. As file size grows toward 100 MB, the TCP/kernel I/O path becomes the
bottleneck and all four implementations converge to roughly the same throughput.
Bash is slightly faster at very large sizes because its built-in `/dev/tcp` +
`cat` pipeline lets the kernel optimize the data path.

## 3. Peak Memory — File Mode

RSS measured while sending a file (not stdin).

| File Size | C | Bash | Python | Ruby |
|-----------|-------:|-------:|---------:|---------:|
| 1 MB      | **1.7 MB** | 3.7 MB |  15.8 MB |  21.0 MB |
| 10 MB     | **1.7 MB** | 3.7 MB |  24.8 MB |  30.0 MB |
| 100 MB    | **1.7 MB** | 3.7 MB | 114.9 MB | 119.9 MB |

**C stays constant at ~1.7 MB** regardless of file size because it streams
file data directly from the file descriptor to the socket — the file is never
loaded into memory. Bash also stays constant (it uses `cat` to pipe the file).
Python and Ruby both `read()` the entire file into an in-memory string, so
their RSS grows linearly with file size.

## 4. Peak Memory — Stdin Mode

All implementations must buffer stdin fully (they need to know the byte count
for the `data:` header before sending).

| Stdin Size | C | Bash | Python | Ruby |
|------------|--------:|---------:|---------:|---------:|
| 1 MB       |  **2.5 MB** |   8.6 MB |  16.5 MB |  21.3 MB |
| 10 MB      | **13.3 MB** |  54.8 MB |  34.3 MB |  30.3 MB |
| 100 MB     | **103.6 MB** | **495.5 MB** | 214.5 MB | 120.3 MB |

- **C** (1.0x input) — one `malloc` buffer holding the stdin data.
- **Ruby** (~1.2x input) — `$stdin.read` returns a single string buffer.
- **Python** (~2.1x input) — holds the read buffer plus encoded bytes.
- **Bash** (~4.9x input) — shell string handling (`data=$(cat; echo x)`) plus
  `echo -ne "$data"` creates multiple internal copies.

## 5. Binary / Script Size

| Implementation | Size on disk |
|----------------|-------------:|
| C (compiled)   |        32 KB |
| C (stripped)    |        20 KB |
| Bash script     |        11 KB |
| Python script   |       8.7 KB |
| Ruby script     |       8.6 KB |

The C binary is larger, but has zero runtime dependencies beyond libc.

## Summary

| Metric | Winner | Runner-up |
|--------|--------|-----------|
| Startup latency | **C** (0.3 ms) | Bash (2.2 ms) |
| Small file throughput | **C** | Bash |
| Large file throughput | Tie (kernel-limited) | — |
| Memory (file mode) | **C** (constant 1.7 MB) | Bash (constant 3.7 MB) |
| Memory (stdin mode) | **C** (1.0x input) | Ruby (1.2x input) |
| Portability | Bash / Python | Ruby |
| Code size | Ruby (230 lines) | Python (280 lines) |
