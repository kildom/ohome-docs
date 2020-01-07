```
FOR NRF51

      | modified vect table | pages ... | original vect table | bootloader + aes_key |
        RESET -> bootloader RESET
        Initial SP -> bootloader Initial SP

      original vect table and bootloader are located  at the region protected by PROTREG63 bit.

Above approach is not safe. App can modify page 0 and make bootloader unusable.
Enrything (bootloader, irq jumps to app and aes_ket) have to be at the beginnig protected by PROTREG0 bit.

IRQ optimization:
Last page of bootloader contains direct jumps to IRQ handlers in app.
Before bootloader starts the app it checks IRQ vectors of the app and compares them with jumps in bootloader.
If there is a difference bootloader erases page and puts a new jumps there.
Each entry in jumps page is fixed size, so vectors in page 0 are fixed.
If verctor in app is unexpected then it falls back to default jump: read app vector value and jump there.

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
erase pages 0..3
                erase done
sendBlock 0
sendBlock 1
...
sendBlock 31 and get status
                ok / missing block bitmap / error
                    missing block bitmap - contains 32 bits for blocks 32*N .. 32*N+31
                    and block in command is in this range. Writing a new block outside
                    this range clears bits internally in device.
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
    output = AES(key = block[0..15], plaintext = output ^ block[16..31])
    
Encryption:
*  Both parts have generated AES key = md5(password & devide unique address)
    * device unique address is placed in 'send catched'
* 'run bootloader' contains AES_CFB encrypted CR = random bytes (different for each connection) and magic bytes
* device genrates random connection id (96 bits) and counter start value (32 bits).
* 'bootloader running' contains AES_CFB (CR, conn_id, counter)
* all further packets will have following structure:
    * lower 2 bytes of counter in plain text
        (higher 2 bytes of counter other end have to predict based on cyclic property of lower 2 bytes)
    * ciphertext = AES_PECB(IV = AES(conn_id & counter), content & zeros)
        (if receiving part gets zeros != 0 then packet was corrupted)
        (total overhead for encryption and authentication of each packet 2 + zeros bytes)
* counter is increased on each transmission (except retransmission)

AES_PECB:
C[0] = AES(IV) ^ P[0]
C[1] = AES(P[0]) ^ P[1]
...
P[0] = AES(IV) ^ C[0]
P[1] = AES(P[0]) ^ C[1]
...

Or better AES_DCTF provides:
      * full data security
      * error propagation
      * can use used instead of both AES_CFB and AES_PECB
      * the same algorithm for encryption and decrition, but with inverted KEYs
AES_DCTF:
AES1(...) with KEY1 = KEY, AES2(...) with KEY2 = KEY ^ 1
C[0] = AES2(IV) ^ T[0] ; T[0] = AES1(IV) ^ P[0]
C[1] = AES2(T[0]) ^ T[1] ; T[1] = AES1(T[0]) ^ P[1]
C[2] = AES2(T[1]) ^ T[2] ; T[2] = AES1(T[1]) ^ P[2]
...
P[0] = AES1(IV) ^ T[0] ; T[0] = AES2(IV) ^ C[0]
P[1] = AES1(T[0]) ^ T[1] ; T[1] = AES2(T[0]) ^ C[1]
P[2] = AES1(T[1]) ^ T[2] ; T[2] = AES2(T[1]) ^ C[2]

Euqlivement:
   AES_DCTF_ENCRYPT(key, iv, plain) = AES_CTF_DECRYPT(key ^ 1, iv, AES_CTF_ENCRYPT(key, iv, plain))
   AES_DCTF_DECRYPT(key, iv, cipher) = AES_CTF_DECRYPT(key, iv, AES_CTF_ENCRYPT(key ^ 1, iv, cipher))
   
OR for better security key ^ 1 can be replaced by some more advanced method, e.g.:
 * key2 = AES(key1, key1)
 * OR assume that connection key is 256-bit long with concatenated two keys:
   * key1 = md5(pwd & devide unique address), key2 = md5("KEY2" & key1)
   * OR key = sha256(pwd & devide unique address)

AES key is located at the end of bootloader's flash page (end of flash). It is programmed the same time as entire bootloader.

```

```c++

void aes_dctf_block(uint8_t* data, size_t size)
{
    do_aes(buffer1); // buffer1 contains key1=key, plain=iv
    do_xor(data, &buffer1[32]);
    memcpy(&buffer1[16], data, 16);
    do_aes(buffer2); // buffer2 contains key2=key^1, plain=iv
    do_xor(data, &buffer2[32]);
    memcpy(&buffer2[16], &buffer1[16], 16);
}

```

```c++

typedef void (*block_callback)(uint8_t* data, size_t size);

uint8_t aes_buffer[16 * 3];

static void do_aes()
{
    ECB->ECBDATAPTR = aes_buffer;
    ECB->TASKS_STARTECB = 1;
    while (!ECB->EVENTS_ENDECB);
    ECB->EVENTS_ENDECB = 0;
}


static void do_blocks(uint8_t* data, size_t size, size_t block_size, block_callback callback)
{
    while (size > block_size)
    {
        callback(data, block_size);
        size -= block_size;
        data += block_size;
    }
    callback(data, size);
}

static void do_xor(uint8_t* data, uint8_t* source, size_t size)
{
    size_t i;
    for (i = 0; i < size; i++)
    {
        data[i] ^= source[i];
    }
}

static void aes_hash_block(uint8_t* data, size_t size)
{
    memcpy(&aes_buffer[0], data, size);
    do_xor(&aes_buffer[16], &aes_buffer[32], 16);
    do_aes();
}

void aes_hash_calc(uint8_t* data, size_t size, uint8_t* hash)
{
    memset(aes_buffer, 0, sizeof(aes_buffer));
    do_blocks(data, size, 32, aes_hash_block);
    memcpy(hash, &aes_buffer[32], 16);
}


void aes_set_key(uint8_t* key)
{
    memcpy(aes_buffer, key, 16);
}

void aes_set_iv(uint8_t* iv)
{
    memcpy(&aes_buffer[16], iv, 16);
}

static void aes_xpecb_encrypt_block(uint8_t* data, size_t size)
{
    do_aes();
    memcpy(&aes_buffer[16], data, size);
    do_xor(data, &aes_buffer[32], size);
}

void aes_xpecb_encrypt(uint8_t* data, size_t size)
{
    do_blocks(data, size, 16, aes_xpecb_encrypt_block);
}

static void aes_xpecb_decrypt_block(uint8_t* data, size_t size)
{
    do_aes();
    do_xor(data, &aes_buffer[32], size);
    memcpy(&aes_buffer[16], data, size);
}

void aes_xpecb_decrypt(uint8_t* data, size_t size)
{
    do_blocks(data, size, 16, aes_xpecb_decrypt_block);
}

// XPECB is actually CFB with inverted encrypt with decrypt process
#define aes_cfb_encrypt aes_xpecb_decrypt
#define aes_cfb_decrypt aes_xpecb_encrypt

```
