
# Architecture

## Threads

| Thread       | Priority | Real Time | Purpose          | Allowed                                                                          |
|--------------|----------|-----------|------------------|----------------------------------------------------------------------------------|
| *interrupts* | IRQ      | yes       | Fast HW response | only minimum to handle IRQ                                                       |
| worker low   | 2        | yes       | Low level logic  | short tasks with <br>deterministic time                                          |
| worker high  | 1        | no        | High level logic | C++11 <br> long running tasks <br> dynamic allocation <br> nondeterministic time |
| idle         | 0        | no        | Put CPU to sleep | N/A                                                                              |
