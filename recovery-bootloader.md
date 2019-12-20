```
FOR NRF51

| modified vect table | pages ... | original vect table | bootloader + aes_key |
  RESET -> bootloader RESET
  Initial SP -> bootloader Initial SP
  
original vect table and bootloader are located  at the region protected by PROTREG63 bit.


[programmer]    [devide]

> start catch user event
send catch
send catch
send catch
send catch
                power on
send catch
send catch
send catch
> stop catch user event
send catch 100
send catch 99
...
send catch 1
send catch 0
                send catched 10
                delay random
                send catched 9
                delay random
                ...
                send catched 0
run bootloader
                bootloader running
> flash user event
erase
                erase done
sendBlock 0
sendBlock 1
...
sendBlock 31 and get status
                ok / missing block bitmap / error
sendBlock 32
...
sendBlock 347 and get status
                ok / missing block bitmap / error
get CRC
                CRC
> run user event
run app
                > soft reset
run app
run app
run app
run app

... OR ...
> flash bootloader user event
sendBlock 0 (to RAM)
sendBlock 1
...
sendBlock 8 and get status
                ok / missing block bitmap / error
get CRC
                CRC
rewrite bootloader
> run user event
run app
run app
run app
run app
run app

Init radio
    2Mbit,
    5 byte addr (MAGIC value, different for each side)
    8 bit LENGTH
    no S0, S1
    3 byte CRC (including address)
    
If bootloader was not catched then do softreset.
    After softreset check reset reason and go directly to the app.

CRC may be replaced by AES based hash (small footprint because of HW AES accelerator):
output[16] = 0
foreach block[32] from input (padding by previous value of block or 0 if input.len < 32)
    output ^= AES(key = block[0..15], plaintext = block[16..31])
    
Encryption:
*  Both parts have generated AES key = md5(password & devide unique address)
    * device unique address is placed in 'send catched'
* 'run bootloader' contains AES_OFB encrypted CR = random bytes (different for each connection) and magic bytes
* device genrates random connection id (96 bits) and counter start value (32 bits).
* all further packets will have following structure:
    * lower 2 bytes of counter in plain text
        (higher 2 bytes of counter other end have to predict based on cyclic property of lower 2 bytes)
    * IV = AES(conn_id & counter)
    * ciphertext = AES_OFB(IV, content)
        (total overhead for encryption and authentication of each packet 2 + 16 = 18 bytes)
* counter is increased on each transmission (except retransmission)
* 'bootloader running' contains input = (CR, conn_id, counter) (all ecrypted using above method)    

AES key is located at the end of bootloader's flash page (end of flash). It is programmed the same time as entire bootloader.

```

