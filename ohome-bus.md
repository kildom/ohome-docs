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
 * Address has to be saved in NV memory and keeped during software update
 * It should be possible to attach special device for electrical diagnostics
 	  * It can send request to each device that forces it to response with some special
	    pattern that can be used to generate eye diagram. This allows to check signal quality at any
	    place on the network from any endpoint.
 * Time is devided into 20ms slots for each device. Device address is a time slot number.
 * There is one additional empty time slot that can be used to add new devices.
   * If device with uninitialized address is attached it listens for incoming packets and sends packet at empty time slot increasing total time slots.
 * There are special packets for network management: adding/removing time slots, chaning device address.
   * They can be used by new device that wants automatically connect to network.
   * Or PC program to manually manage the network.

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


Discovery options (not up to date)
----------------

### Option 1

* Autodiscovery process (trigrred by the user to save overhead during normal operation):		
	* Master periodically sends 0xFF and END (start transaction with device 0x7F) and waits 25ms	
	* Unconfigured device responses with higher level packet containing device uniue id	
		* It is randomly determined how many 0xFF's device waits until response
		* Response time is randomly chosen between 0 and 20ms (problem: such packet may be 10ms length)
		* If some other node started response before then skip responding
	* Master sends to address 0x7F higher level packet containing device unique id and newly assigned address.	

### Option 2

**Master**
* Sends "start discovery" packet and waits 30ms		
	* If there is response then continue with discovery process	
* Send "check bit" packet for each bit		
	* If response ended with END is received then stop with bit checking	
* Send "assign" packet to assign address to unconfigured device		
* Check if configuration was correct by quering transfer parameters		
* Repeat the process until all devices were discovered		

**Slave**
* If packet with ADDR=0x7F was received:		
	* switches to level measuring: if low level detected then stop further steps	
	* update pattern ("check bit") or reset pattern ("start discovery")	
	* if pattern does not match unique id then stop at this point	
	* if pattern has no "x" then:	
		* send END
		* take a new address (only "assign" packet)
		* stop at this point
	* waits 25ms to make sure that all slow devices switched	
	* drive line low for 3 bits period (or send 0xFC)	

Packets:

|start discovery:|	check bit:|	assign:
|------|-------|-------|
|ADDR = 0x7F|	ADDR = 0x7F|	ADDR = 0x7F|
|END	|bit value 0..1|	127|
| |position 0..127|	new address|
| |	END|	END|

Example:

|packet|	matching pattern|	response|
|------|------------------|---------|
|start|	xxxxxxxxxxxxxxxx|	yes|
|0@0|	0xxxxxxxxxxxxxxx|	yes|
|0@1|	00xxxxxxxxxxxxxx|	yes|
|0@2|	000xxxxxxxxxxxxx	|yes|
|0@3|	0000xxxxxxxxxxxx	|yes|
|0@4|	00000xxxxxxxxxxx	|no|
|1@4|	00001xxxxxxxxxxx	|yes|
|0@5|	000010xxxxxxxxxx|	yes|
|0@6|	0000100xxxxxxxxx|	no|
|1@6|	0000101xxxxxxxxx|	yes|
|...	|...|	...|
|1@15	|000010100010011	|END|
|assign	|000010100010011	|END|

### Option 3

Similar to Option 2

Packets:

|start discovery: |	init: |	next bit: |	assign: |
|------------|----|----|----|
|ADDR = 0x7F|	ADDR = 0x7F	| ADDR = 0x7F|	ADDR = 0x7F|
|END	|2	|prev bit 0..1|	3|
| |	END|	END	|new address |
| |	|	|	END|


Example:

|	packet | 	matching pattern | 	response | 
|--------|-------------------|--------|
|	start | 	xxxxxxxxxxxxxxxx | 	yes | 
|	2 - init | 	xxxxxxxxxxxxxxxx | 	0 | 
|	0 | 	0xxxxxxxxxxxxxxx | 	0 | 
|	0 | 	00xxxxxxxxxxxxxx | 	0 | 
|	0 | 	000xxxxxxxxxxxxx | 	0 | 
|	0 | 	0000xxxxxxxxxxxx | 	1 | 
|	1 | 	00001xxxxxxxxxxx | 	0 | 
|	0 | 	000010xxxxxxxxxx | 	1 | 
|	1 | 	0000101xxxxxxxxx | 	0 | 
|	0 | 	00001010xxxxxxxx | 	0 | 
|	... | 	... | 	... | 
|	1 | 	000010100010011 | 	END | 
|	3 - assign | 	000010100010011 | 	END | 

> "END" may conflict with "1" from device with the same beginning of UID, but longer
>
> Solution 1: prefix UID with its length (only for discovery process)
