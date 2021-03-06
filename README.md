# Modbus communication library

This library provides functions for serial communication. Runs only on an
FPGA (Field Programmable Gate Array). Used mainly to communicate on a
coordinator-worker bus, usually with indexable daisy chained workers.
[Modbus](http://modbus.org) or other protocol can be used.

:warning: **[CRC16](https://crccalc.com) checksum calculation is not
included**. The calling process should calculate and include the transformed
checksum as data payload, and check received data CRC16 (or any other CRC).

## Physical interface

A serial interface is constructed from two DIO (Discrete Input/Output) pins.
One pin has to be configured for input (receiving), the other for output
(transmitting).  This serial communication is often implemented using a
[NI-9401 Digital Module](https://www.ni.com/en-us/support/model.ni-9401.html).
This provides 8 digital I/O pins, which can be used as 4 input and 4 output
ports.

## LabView/Host interface

[Common_FPGA_HealthAndStatus
library](https://github.com/lsst-ts/Common_FPGA_HealthAndStatus/) is used for
health and status FIFO(First In First Out). That provides datatype for the FIFO.

For each use, one input and one output DIO are needed. Six FIFOs has to be
provided - transmitting, receiving, timestamps, health & status for Rx and Tx,
and internal transmitting queue. Three boolean registers are needed - IRQ,
WaitForIRQ, and ModbusTrigger (starting physical write after receiving 0x8000
"Wait for trigger command").

Each FIFO should be configured to "never arbitrate" for operation used inside
the library. Never arbitrate on write should be set for output/receiving (with
write method called inside the library). Never arbitrate on read should be set
for input/transmission (with read method called inside the library). The best
is to leave both write and read as "never arbitrate".

For each use:

1. Create a U16 FIFO for received data.

2. Create a U16 FIFO for transmitted data.

3. Create a custom type ModbusTxInstruction FIFO for internal usage while
   transmitting data. Specify never arbitrate for read and write.

4. Create two custom type HealthAndStatusUpdate FIFOs - one for receiving,
   second for transmission.

5. Create a boolean register for IRQ control and "WaitForRXFrame". Specify
   never arbitrate for write.

6. Create a boolean register for the software controlled trigger. Specify never
   arbitrate for write. You can use a single trigger for controlling multiple
   ports so this is an optional step if a register has already been generated.

7. Create a FIFO for Tx timestamps. Specify a data type of U64. Specify never
   arbitrate for write.

8. Drop the FPGAModbus/ModbusPort.vi onto your main VI.

9. Setup the ModbusPort settings using the resources defined above.

10. Sets base address (shall be multiply of 6) and IRQ (ordinals).

11. Review the comments in ModbusPort.vi for more information.

### Optional

To copy health and status data FIFOs into a single FIFO:

12. Drop the FPGAHealthAndStatus/HealthAndStatusFIFOCopy.vi. Set the internal
   health and status FIFO. Set the source as the FIFO generated in step 2.

13. Drop the FPGAHealthAndStatus/HealthAndStatusFIFOCopy.vi. Set the internal
    health and status FIFO. Set the source as the FIFO generated in step 5.

# How it works

FIFOs are used to provide a channel between
variable speeds write and read ends. Usually one end of the FIFO is inside
[SCTL](https://knowledge.ni.com/KnowledgeArticleDetails?id=kA00Z000000P8sWSAS&l=en-US).
SCTL are used to grant execution of the commands inside FPGA in a single tick.

Link speed is hard-coded to 44 40Mhz ticks per bit, which roughly equal to
921600 bauds *(actual number is 40M / 44 = 909090.909; but as delays are
inserted between frames, and frames aren't long, 44 works, sampling a slightly
different part of receiving signal for each received bit)*.

One start and one stop bits are written (must be included in data, see below).

Transmitting FIFO is U16 typed. The most significant nibble is used for
instruction. Nibbles 2-4 are used for data.

Data from transmitting FIFO are parsed into instructions and data, and placed
into internal FIFO. Instructions are then processed from the internal FIFO -
see below for instructions types. Proper signal (including frame start, start
bit, data, stop bit, frame delay) is generated.

Received line is checked for frame start. Bytes are then stored into receiving
FIFO. Frame end is detected on line silence. IRQ is raised when frame is
received. As the expected use is in a coordinator-worker communication, IRQ
doesn't have to be handled outside ModbusPort subVI. Data is written into the
receiving
FIFO.

CRC16 isn't checked on received FIFO and the CRC16 (if part of the
communication) is stored into receiving FIFO.

# Instructions

Opcodes 1-8 are used for transmission (Tx FIFO). Instructions 9 and 10 are used
in received response (Rx FIFO).

 OpCode | Description
 ------ | -----------
 0x1    | sends data. *Data* are in bits 9-0, and includes start and stop bits. The least significant bit (bit 0) is a start bit, must be 0. 8 data bits follow. Bit 9 is stop  bit, must be 1. To write 8 bit data, shift left (with 0 fill) by 1, or result with 0x0200
 0x2    | end of frame, delay in ticks (data). Should be 0xda *(/44 =~ 5)*, resulting in 0x20da written
 0x3    | push TimestampRegister to Tx timestamps FIFO
 0x4    | delay for *data* microseconds
 0x5    | delay for *data* milliseconds
 0x6    | wait for a received frame. Data = timeout in us (microseconds, 1/1M of a second, 40 ticks). Recommended value is 0x3e8 = 1ms
 0x7    | triggers IRQ
 0x8    | wait for trigger (ModbusTrigger == true)
 0x6    | long wait for a received frame. Data = timeout in ms (milliseconds, 1/1K of a second, 40000 ticks)
 0x9    | received *data* (on received FIFO)
 0xA    | received end of frame (on received FIFO)
 0xB    | timestamp U8; 8 0xB are expected, contain low endian timestamp from FPGA clocks

For synchronization with the other modbus ports, a frame should be prefixed
with 0x8 command. Data wouldn't be transmitted before ModbusTrigger == true.
ModbusTrigger shall be turned to false by process/subVi calling 0x8 command.

## Timestamps

Timestamps are taken from the TimestampRegister parameter. It's responsibility
of calling Vi to set and update the register. See
[Timestamp.vi](https://github.com/lsst-ts/ts_M1M3SupportFPGA/blob/master/src/Timestamp/Timestamp.vi)
for possible implementation.

## Example

As an example, to call Modbus function 17 = 0x11 for address 0x81, so the
payload is:

```
0x81 0x11
```

with CRC:

| Serv. Addr | Func. Code | CRC   |
|------------|------------|-------|
| 81         | 11         | A1 EC |

*[CRC16/MODBUS](https://crccalc.com) of 0x81 0x11 = 0xeca1, transmitted as low
endian (hence 0xa1 0xec, right shifted by 1 = 0x142 0x1d8; with stop bit =
0x342 0x3d8).*

Send the following commands:

* set WaitForTrigger
* push current timestamp to Tx timestamp FIFO; timestamp will be pushed after trigger is released
* write payload, CRC and frame end
* wait 1ms for reply
* signal TriggerIRQ

To send these commands write the following 2-byte (U16) numbers into the
transmitting FIFO.

```
0x8000 0x3000 0x1302 0x1222 0x1342 0x13d8 0x20da 0x63e8 0x7000
```

Assuming the device returns the following data:

| Serv. Addr | Func. Code | Length | Unique ID         | ILC AppType | Net. Node | ILC Sel.Opt. | Net.Node Options | Maj. Rev. | Min. Rev. | Firmware Name    | CRC   |
| ---------- | ---------- | ------ | ----------------- | ----------  | --------- | ------------ | ---------------- | --------- | --------- | ---------------- | ----- |
| 81         | 11         |   10   | 12 34 56 78 90 AA |   FF        |        BB |   CC         |   DD             |        EE |        11 | 53 74 61 72      | A7 9F |

this shall be read from the receiving FIFO:

```
0x9302 0x9222 0x9220 0x9224 0x9268 0x92ac 0x92f0 0x9320 0x9354 0x93fe 0x9398 0x93ba 0x93dc 0x9222 0x92a6 0x92e8 0x92c2 0x92e4 
0x934e 0x933e
0xB077 0xB066 0xB055 0xB044 0xB033 0xB022 0xB011 0xB000
0xA000
```

Where:

* response bytes (0x9) payload, with start/stop bites (check for
  (response _(U16)_ and _(bitwise)_ 0x0201) = 0x0200, right shift by 1 and mask
  with 0xFF to get data)
* last two 0x9... are CRC16 bytes (for the payload), see above for shifting
* following 0xB... are 8 16 bit numbers of low endian 64 bit timestamp.
  Bitwise with 0xFF to get the value. Timestamp is taken from TimeStampRegister
  when failing edge is detected (start of receiving the data). The example
  above depicts number 0x0011223344556677 = dec 4822678189205111
* last 0xA000 signal frame ends

:warning: It is reader's responsibility to check CRC16. No checking is done
inside the library.

# Health and Status telemetry

Health and status telemetry is provided as instructions written into Health and
Status FIFO. Please see [Common_FPGA_HealthAndStatus
library](https://github.com/lsst-ts/Common_FPGA_HealthAndStatus/) for details.

The following offsets from the base address specified in ModbusPort settings
are populated:

| Offset| Description                                                             |
|-------|-------------------------------------------------------------------------|
| **0** | Error flags. See below for details                                      |
| **1** | transmitted (Tx) byte count                                             |
| **2** | transmitted (Tx) frame count                                            |
| **3** | received (Rx) byte count                                                |
| **4** | received (Rx) frame count                                               |
| **5** | transmitted (Tx) instructions count; shall be higher than Tx byte count |

## Error Flags

| Bit | Value |  Hex |  Name                         |
| --- | ----- | ---- | ----------------------------- |
|  0  |  1    | 0001 | TxInternalFIFO Overflow       |
|  1  |  2    | 0002 | Invalid Instruction           |
|  2  |  4    | 0004 | Wait For Rx Frame Timeout     |
|  3  |  8    | 0008 | Start Bit Error               |
|  4  |  16   | 0010 | Stop Bit Error                |
|  5  |  32   | 0020 | RxDataFIFO Overflow           |
|  6  |  64   | 0040 | Waiting for software trigger  |
