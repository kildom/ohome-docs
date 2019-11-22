Device configuration commands
=============================

### Get info

| Response | | |
|-|-|-|
| byte | address | address is assigned once during attaching<br>to the network and cleared when detaching |
| byte | UID length |
| byte[] | UID | starts with one byte family ID and rest is derectly optained from microcontroller |
| string | name |
| string | architecture |
| byte[20] | software version | git commit hash in binary format |

### Set name

| Request | | |
|-|-|-|
| string | new name |

### Get stack usage

| Response | | |
|-|-|-|
| string | thread name |
| uint32_t | total stack bytes |
| uint32_t | free stack bytes |
| ... | | Repeated for each thread, i.e. *main*, *low*, *high* |

### Get CPU load

| Response | | |
|-|-|-|
| uint32_t | total ticks |
| uint32_t | idle ticks |
| uint32_t | low max delay |
| uint32_t | low max execution |
| uint32_t | high max delay |
| uint32_t | high max execution |
| | | *(if app is not compiled with specific feature<br>then value is -1)* |

Network management commands
===========================

### Start wire discovery

| Responses (multiple) | | |
|-|-|-|
| byte | address |
| byte[] | UID |
| | | *(and empty response at the end<br>of the discovery)* |

### Get wire transmission parameters

| Response | | ***Unsupported*** if node does not use wire transmission |
|-|-|-|
|uint8_t|	block delay|
|uint8_t|	start delay|
|uint8_t|	max response time|
|uint8_t|	block size| skipped if *block delay* == 0 |

