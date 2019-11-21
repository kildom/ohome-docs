
Network management commands
===========================

### Read UID

| Response | | |
|-|-|-|
| byte[] | UID | starts with one byte family ID and rest is derectly optained from microcontroller |

### Read wire transmission parameters

| Response | | |
|-|-|-|
|uint8_t|	block delay|
|uint8_t|	start delay|
|uint8_t|	max response time|
|uint8_t|	block size| skipped if *block delay* == 0 |

