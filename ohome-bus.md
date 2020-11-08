ohome bus
=========

This darft describes how to transfer packets over a wires.
It also defines how to handle device addres discovery.
This protocol is used for communication between network nodes.
There is no special node during normal communication.
Address discovery is started by the user on one of network nodes.
This node becomes discovery controller for time of address discovery.

Higher level protocol is described somewhere else. The same higher level protocol is used for radio transmission.

General notes
-------------

 * Electrical interface: RS-485 with multiple nodes, half duplex	
 * Signaling: UART 8-bit, no parity, 115200 (or slower if it is too fast)
 * It should be possible to attach special device for electrical diagnostics
 	  * It can send request to each device that forces it to response with some special
	    pattern that can be used to generate eye diagram. This allows to check signal quality at any
	    place on the network from any endpoint.

Sending
-----------
* Just before sending (as close as possible) device must check if line is free. If not it goes into unsychnonized mode.
* It sends information:
  * When it wants to send next packet
  * When is the closest scheduled packet

Receiving
---------
* If following test fail then go to unsynchronized mode:
  * Check if closest scheduled packet is matching expected
  * Check if next packet is not overlapping with currently scheduled
  
Unsychronized mode
------------------


Packet format
-----------

| Length | Value |
|--------|-------|
| 1 | Start byte 0x9B |
| 1 | Replacement byte |
| 1 | Source address |
| 1 | Total known addresses |
| 1 | Flags |
|   | 0 - this is management packet |
|   | 1 - give rest of timeslot to different device |
| 1* | Next device address (if flag 1) |
| 1 | Length |
| 0..246 | Data |
| 2 | CRC-16 |
