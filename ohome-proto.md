
Network management commands
===========================

### Start wire discovery

| Multiple responses | | |
|-|-|-|
| byte | address |
| byte[] | UID |

| Final response | | |
|-|-|-|
| byte |  |
| byte[] | UID |

### Read UID

| Response | | |
|-|-|-|
| byte[] | UID | starts with one byte family ID and rest is derectly optained from microcontroller |

### Read wire transmission parameters

| Response | | ***Unsupported*** if node does not use wire transmission |
|-|-|-|
|uint8_t|	block delay|
|uint8_t|	start delay|
|uint8_t|	max response time|
|uint8_t|	block size| skipped if *block delay* == 0 |

