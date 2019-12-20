```
FOR NRF51

| modified vect table | pages ... | original vect table | bootloader |
  RESET -> bootloader RESET
  Initial SP -> bootloader Initial SP


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

```

