# Svive Triton RGB TKL Configuration

This is an attempt to document the protocol used by the **Svive Triton TKL connect** software to configure things like key bindings, macros and light settings on the **Svive Triton RGB TKL** keyboard.

The software does not appear to read configuration from the device.

The device shows up as a SONiX USB DEVICE 0C45:8006.

## Messages
The software initiates all requests by sending a 64 byte message and then reading a 64 byte response.

This initial message pair starts with a two byte *OpCode*.

If the initial response indicates success read or write operations for data follow for operations that transfer data.

Data transfer is always followed by a `04 02` operation to get the checksum of the payload data (from the keyboards point of view) to verify data integrity.

### Request Message
The request message contains the OpCode and number of data pages (64 byte each) to transfer.

| Offset | Length | What
|:-------|:------|:----
| 0x00   | 2     | OpCode
| 0x08   | 1     | Number of data pages to transfer

### Response Message
For most requests the response is identical to the request except that byte 3 is set, presumably to indicate success. The response to `04 02` is an exception and contains a 32 bit checksum of the most recently transferred data.

| Offset | Length | What
|:-------|:-------|:----
| 0x00   | 2      | OpCode
| 0x03   | 1      | Status
| 0x04   | 4      | Checksum (OpCode `04 02` only)
| 0x08   | 1      | Number of data pages to transfer




### Operations
|  OpCode  |  What  |  Request Data  | Response Data
|:--------|:------|:--------------|:--------------
|`04 02`| Request checksum of previous data payload || OK + checksum value
|`04 05` | Start read Info | Number of 64 byte blocks to follow (2, read) | OK
|`04 11` | Start keymap upload | Number of 64 byte messages to follow (10)| OK
|`04 13` | Start color settings upload | Number of 64 byte messages to follow (18) | OK
|`04 15` | Start macro upload | Number of 64 byte messages to follow | OK
|`04 17` | Start *gaming mode* upload | Number of 64 byte messages to follow (1) |OK
|`04 18` | ? || OK
|`04 19` | ? || OK
|`04 ab` | Start random data upload || OK

## Checksums
There are two different types of checksums returned by the `04 02` operation.

Note that the checksum values returned from the keyboard are big-endian.

### Checksum type 1
The first checksum format is all the bytes of the payload data added together into an unsigned 32bit integer.
```
uint8_t data[] = { data };
uint32_t checksum = data[0] + data[1] + ...
```

This is what we get after most data transfer operations.

### Checksum type 2
For the `04 AB` transfer operations the checksum is calculated differently.
```
uint8_t b1 = data[12] + data[3];
uint8_t b2 = data[63] * 8;
uint8_t b3 = data[6] - data[13];
uint8_t b4 = data[36] & 0xa5;
uint32_t checksum = b1 << 24 | b2 << 16 | b3 << 8 | b4;
checksum ^= 0x54711688;
```


## Payload data
Payload data is read or written.

### 0405 - Info data
| Offset | Length | What
|:-------|:-------|:----
| 0x00   | 1      | ? (0x40)
| 0x01   | 1      | ? (0x30)
| 0x02   | 2      | ? (0x00)
| 0x04   | 2      | USB idVendor
| 0x06   | 2      | USB idProduct
| 0x08   | 2      | USB bcdDevice
| 0x10   | 2      | ? (0xFF)
| 0x12   | 110    | ? (0x00)

### 0411 - Keymap data
This is a lookup table with mappings for each key on the keyboard, 4 byte per key.

| Index | Key
|:------|:----
| 0     |
| 1     | ESC
| 2     | F1
| 3     | F2
| 4     | F3
| 5     | F4
| ...   | ...

The FN key can be remapped, so that it no longer works as the FN key, but I don't know what to write if I want to make another key the FN key.

Examples of mappings for a key

|  Action           | Bytes
|:------------------|:------
| Default           | `00 00 00 00`
| Multimedia e-mail | `00 00 05 03`
| Mouse left        | `01 01 01 00`
| Mouse right       | `01 01 02 00`
| Mouse middle      | `01 01 04 00`
| Mouse scroll up   | `01 03 01 00`
| Mouse scroll down | `01 03 ff 00`
| Key bind          | `02 00 key 00`
| Win hotkey        | `02 01 key 00`
| Run program       | `05 02 01 00` ?
| Forbidden         | `05 03 00 00`
| Run macro         | `06 id 01 repeat_num`
| Multi key         | `07 key1 key2 key3`

Key is a [USB HID usage code](https://www.usb.org/sites/default/files/documents/hid1_11.pdf).

### 0413 - Color data
A 1152 bytes payload spread over 18 writes.
- Effect settings, 32 effects 16 bytes/effect
- Key colors for when custom colors is selected (144*4b)
- Active effect (16 bytes, repeated from list above)

| Offset | Length | What
|:-------|:-------|:----
| 0x00   | 16     | Effect 1
| 0x10   | 16     | Effect 2
| ...    | ...    |
| 0x1F0  | 16     | Effect 32
| 0x200  | 4      | Key 0 color
| 0x204  | 4      | Key 1 (ESC) color
| ...    | ...    |
| 0x440  | 16     | Active effect


#### Effect settings

| Offset | Length | What
|:-------|:-------|:----
| 0x00   | 1      | ID
| 0x01   | 1      | R
| 0x02   | 1      | G
| 0x03   | 1      | B
| 0x04   | 4      | ? (all zero)
| 0x08   | 1      | Full color
| 0x09   | 1      | Brightness
| 0x0A   | 1      | Speed
| 0x0B   | 1      | Direction
| 0x0C   | 4      | ? (00 00 AA 55)

#### Key colors
| Offset | Length | What
|:-------|:-------|:----
| 0x00   | 1      | ? (mostly 0x80)
| 0x01   | 1      | R
| 0x02   | 1      | G
| 0x03   | 1      | B

### 0415 - Macro data
This is not a fixed size message, size will grow to accommodate more macros/macro-steps.

The data starts with 100 32bit offsets to macro data, relative to the start of the message.

Each macros start with the number of steps (64bit) in the macro followed by 4 bytes per step.

| Offset | Length | What
|:-------|:-------|:----
| 0x00   | 4      | Offset to macro 1 data
| 0x04   | 4      | Offset to macro 2 data
| ...    | ...    | ...
| 0x18C  | 4      | Offset to macro 100 data
| 0x190  | ...    | Macro data

#### Macro
| Offset | Length | What
|:-------|:-------|:----
| 0x00   | 8      | Number of steps
| 0x08   | 4      | Step 1
| 0x0C   | 4      | ...


##### Macro steps
Each step in a macro is represented with 32 bits.
Values are not necessarily full bytes.

| What       | Bits
|:-----------|:-----
| Delay      | 5 bits (0xA), 27 bits (number of milliseconds)
| Key down   | 5 bits (0x16), key (usage code)
| Key up     | 5 bits (0x06), key (usage code)
| Mouse down | 5 bits (0x12), button
| Mouse up   | 5 bits (0x02), button

### 0417 - 'Gaming mode' data
| Offset | Length | What
|:-------|:-------|:----
| 0x00   | 1      | ? (0xA)
| 0x01   | 1      | Enable gaming mode?
| 0x02   | 1      | Disable `Alt + Tab`
| 0x03   | 1      | Disable `Alt + F4`
| 0x04   | 1      | Disable `WinKey`
| ...    | ...    |
| 0x3E   | 2      | AA 55

### 04AB - Random data
This appears to be 64 bytes of random data sent to the keyboard.

It could be used for the random colors (full colors) for effects?
Or maybe it is just for testing that transferring data is working.
