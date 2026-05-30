---
title: ROM Programmer
date: 2026-05-29
categories: [Breadboard CPU]
description: Design and code for Arduino-based flash programmer
---

## Background and terminology

The primary use of read-only media (ROM) in the CPU design is to store large amounts of data that do not need to vary during normal operation. This is especially beneficial when replacing combinational logic; for example, "translating" a string of bits into a representation that can be shown on a 7-segment numerical display.

The SAP-1 uses an electronically erasable programmable ROM (EEPROM) for this purpose, while the F8-BB uses its UV-erasable counterpart (UV EPROM). In the ASAP-3, I use the SST39SF040, which uses NOR flash instead of EPROM, because I could not source inexpensive through-hole EEPROM chips and (frankly) did not want to deal with UV-light erasure. At the level of abstraction of this project, the differences between EPROM and flash technology are negligible. However, programming these chips presented some interesting challenges, which I will describe below.

## Making life easier with the Arduino Mega



## Handling software data protection

Unlike the EPROMs, the SST39SF040 has several hardware and software data protection features meant to avoid inadvertent write operations or other data loss. Most importantly, multi-byte command sequences are required to write to or erase from the chip:

### Table 1. Byte-program command sequence

| Write Cycle | Address | Data |
| :---: | :---: | :---: |
| 1 | 0x5555 | 0xAA |
| 2 | 0x2AAA | 0x55 |
| 3 | 0x5555 | 0xA0 |
| 4 | **Address** | **Data** |

### Table 2. Chip-erase command sequence

| Write Cycle | Address | Data |
| :---: | :---: | :---: |
| 1 | 0x5555 | 0xAA |
| 2 | 0x2AAA | 0x55 |
| 3 | 0x5555 | 0x80 |
| 4 | 0x5555 | 0xAA |
| 5 | 0x2AAA | 0x55 |
| 6 | 0x5555 | 0x10 |

I define these sequences in the header of my Arduino code:

```c++
const int PGM_CYCLES = 3;
const int PGM_CYCLE_ADDR[] = {0x5555, 0x2AAA, 0x5555};
const byte PGM_CYCLE_DATA[] = {0xAA, 0x55, 0xA0};
const int ERASE_CYCLES = 6;
const int ERASE_CYCLE_ADDR[] = {0x5555, 0x2AAA, 0x5555, 0x5555, 0x2AAA, 0x5555};
const byte ERASE_CYCLE_DATA[] = {0xAA, 0x55, 0x80, 0xAA, 0x55, 0x10};
```

Additionally, the timing diagrams for these command sequences are somewhat stricter than those used for the EPROMS. The two most relevant parameters, *T<sub>BP</sub>* and *T<sub>SCE</sub>*, are listed below. Strictly controlling for the other parameters (which are <100 ns minimum) is unnecessary.

### Figure 1. WE_controlled byte-program timing diagram

![a timing diagram](/assets/images/pgm-cycle-td.png)

### Figure 2. Chip-erase timing diagram

![a timing diagram](/assets/images/chip-erase-td.png)


### Table 3. Timing parameters

| Parameter | Min | Max |
| :---: | :---: | :---: |
| Byte-program time (T<sub>BP</sub>) | | 20 &micro;s |
| Chip-erase time (T<sub>SCE</sub>) | | 100 ms |

My Arduino function for writing to the chip, then, first writes the 3 byte-program commands, then writes the actual data byte, then waits for the byte-program time of 20 &micro;s to give the chip enough time to execute the write operation.

```c++
void write_FLASH(byte data[], size_t size, long address_offset = 0) {
  
  digitalWrite(FLASH_OE, HIGH); // Unset output_enable

  // Configure FLASH data pins as outputs
  for (int pin = FLASH_D0; pin <= FLASH_D7; pin++) {
    pinMode(pin, OUTPUT);
  }
  
  for (long address = address_offset; address < size + address_offset; address++) {
    // FLASH chip write command sequence
    for (int cycle = 0; cycle < PGM_CYCLES; cycle++) {
      write_byte(PGM_CYCLE_ADDR[cycle], PGM_CYCLE_DATA[cycle]);
    }

    // Write data
    write_byte(address, data[address - address_offset]);
    delayMicroseconds(20); // byte program time
  }
}
```

Similarly, my erase function writes the 6-byte erase sequence, followed by a 100 ms delay to allow time for the chip-erase execution:

```c++
void erase_FLASH() {
  
  digitalWrite(FLASH_OE, HIGH); // Unset output_enable

  // Configure FLASH data pins as outputs
  for (int pin = FLASH_D0; pin <= FLASH_D7; pin++) {
    pinMode(pin, OUTPUT);
  }

  // FLASH chip erase sequence
  Serial.print("Erasing FLASH...");
  for (int cycle = 0; cycle < ERASE_CYCLES; cycle++) {
    write_byte(ERASE_CYCLE_ADDR[cycle], ERASE_CYCLE_DATA[cycle]);
  }

  delay(100); // chip erase time
  Serial.println(" done.");
}
```

## Serial protocol for transmitting data

The second and bigger hurdle to overcome was simply *getting the data onto the Arduino*. Since the Arduino Mega has 256 KB of flash memory, it cannot store the 4 MB ROM image all at once. I was interested in solving this problem without an SD card interface or dedicated EPROM programmer device, so I decided to develop a method to send serial data to the Arduino.
