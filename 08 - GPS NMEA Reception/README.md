# 08 - GPS NMEA Reception

Saved: 2026-08-03

Hardware validation status: firmware builds successfully; physical UART and
raw-NMEA reception test is pending.

## Goal

Receive complete raw NMEA sentences from a GY-NEO6MV2 breakout containing a
u-blox NEO-6M GPS receiver, then forward them to the PC through the Nucleo
ST-LINK Virtual COM Port.

This stage proves power, ground, crossed UART wiring, baud rate, interrupt
delivery, message framing, buffering, and error reporting. It deliberately does
not parse coordinates or claim that a GPS fix is valid.

## Connections

| GPS breakout | NUCLEO-F334R8 | Function |
|---|---|---|
| VCC | 5V | Breakout input through its onboard regulator |
| GND | GND | Common electrical reference |
| TX | PA10 / Arduino D2 | USART1 receive |
| RX | PA9 / Arduino D8 | USART1 transmit for later configuration |

Serial lines cross because TX and RX names are relative to each device:

~~~text
GPS TX  ---->  STM32 USART1_RX
GPS RX  <----  STM32 USART1_TX
~~~

The onboard ST-LINK terminal remains on USART2:

~~~text
PA2 = USART2_TX, 38400 baud
PA3 = USART2_RX, 38400 baud
~~~

Do not connect GPS TX to PA3. Two devices driving the same serial path would
mix independent signals and make debugging misleading.

## UART configuration

USART1 is configured by CubeMX as:

~~~text
Asynchronous mode
9600 baud
8 data bits
No parity
1 stop bit
No hardware flow control
16x oversampling
USART1 global interrupt enabled
~~~

The NEO-6M default NMEA configuration uses 9600 baud. Both endpoints must use
the same frame format. A baud mismatch changes when each bit is sampled,
usually producing garbage characters, framing errors, or no valid lines.

## What asynchronous serial means

UART sends no shared clock wire. Each byte is framed on one signal:

~~~text
idle | start bit | 8 data bits | stop bit | idle
~~~

At 9600 baud, one bit takes approximately 104 microseconds. A complete 10-bit
8-N-1 frame takes about 1.04 milliseconds, so the theoretical byte rate is
approximately 960 bytes per second.

## Why interrupt-driven reception is used

A blocking receive call would make the CPU wait for GPS data. That is simple
but prevents the main loop from doing other time-sensitive work.

GPS_StartReception() instead calls:

~~~c
HAL_UART_Receive_IT(&huart1, &gps_receiver.received_byte, 1U);
~~~

The function returns immediately. When one byte arrives:

1. USART1 raises an interrupt.
2. USART1_IRQHandler() calls the STM32 HAL handler.
3. HAL calls HAL_UART_RxCpltCallback().
4. The callback processes the byte.
5. The callback arms reception for the next byte.

Receiving one byte per interrupt is easy to understand and sufficient at 9600
baud. A future higher-rate implementation can use circular DMA.

## How NMEA sentences are assembled

NMEA navigation messages are ASCII text. A normal sentence resembles:

~~~text
$GPGGA,...
$GPRMC,...
~~~

The callback waits for a dollar sign, which provides synchronization even if
the STM32 starts listening halfway through an existing transmission. It ignores
carriage return and finishes the sentence on newline.

Bytes are stored only while there is capacity:

~~~c
if (building_length < GPS_NMEA_LINE_SIZE - 1)
{
    building_line[building_length++] = byte;
}
else
{
    overflow_lines++;
    discard_the_sentence;
}
~~~

The reserved final byte stores the C string terminator. Without this bound
check, one long or corrupted message could write beyond the array and damage
unrelated memory.

## Why there are two buffers

The interrupt writes building_line while a sentence is arriving. Once newline
arrives, it copies that sentence to ready_line.

~~~text
USART1 interrupt -> building_line -> ready_line -> main loop -> USART2
~~~

The interrupt can immediately receive the next sentence while main prints the
previous one. UART transmission is intentionally not performed inside the
interrupt because blocking inside an interrupt increases latency.

## Shared data and critical sections

The interrupt and main loop both access the ready buffer and status counters.
Fields that can change asynchronously are declared volatile, telling the
compiler that their values must be reread rather than assumed unchanged.

GPS_TakeCompletedLine() briefly disables interrupts while copying the ready
sentence and restores the previous interrupt state afterward.

Volatile does not make a multi-byte copy atomic. The critical section prevents
the interrupt from changing shared state halfway through the copy.

## Health counters

Every five seconds, GPS_PrintStatus() reports:

- bytes: all bytes received electrically on USART1.
- lines: complete NMEA sentences assembled.
- dropped: lines discarded because main had not consumed the previous line.
- overflow: sentences longer than the fixed buffer.
- errors: UART framing, noise, overrun, or restart failures.
- last_byte: elapsed milliseconds since the last received byte.

| Result | Likely interpretation |
|---|---|
| bytes=0 | Power, ground, TX wiring, wrong pin, or silent module |
| Bytes increase, lines stay zero | Wrong baud or malformed/non-NMEA stream |
| Lines increase | UART and NMEA framing work |
| Dropped increases | Main loop is not consuming fast enough |
| Overflow increases | Buffer too small or corrupted framing |
| Errors increase | Noise, baud mismatch, weak ground, or wiring integrity |

## Expected terminal output

~~~text
08A - GPS raw NMEA reception
GPS USART1: PA10/RX, PA9/TX, 9600 baud, 8-N-1
Console USART2: ST-LINK VCP, 38400 baud, 8-N-1
USART1 interrupt reception armed; waiting for '$' NMEA sentences.

GPS RAW: $GPGGA,...
GPS RAW: $GPRMC,...
GPS STATUS: bytes=... lines=... dropped=0 overflow=0 errors=0 ...
~~~

Raw sentences should appear even before a navigation fix. A missing outdoor fix
does not explain bytes=0; it only changes validity fields inside the sentences.

## Test conditions

- Put the patch antenna ceramic face upward.
- Use open sky for the later position-fix test.
- Keep the tiny RF connector and cable free from pulling or twisting.
- Confirm the module does not become hot after power is applied.
- Breakout LED behavior varies between board clones and is not proof of UART.

## What is not implemented yet

- NMEA XOR checksum verification.
- GGA/RMC parsing.
- Latitude/longitude conversion.
- Fix-validity decisions.
- UBX binary configuration.
- GPS and IMU integration.

Those are intentionally separate stages. Stage 08 answers only: can the STM32
safely and continuously receive complete GPS text messages?

## Separation from the IMU project

The exact IMU development remains preserved in versions 04 through 07,
including their complete project ZIP files. CubeMX disabled I2C1 for this
standalone GPS test, and the active stage-08 source contains no ICM-20948 or
AK09916 code. Keeping this file GPS-only makes the hardware path, interrupts,
buffers, and test results much easier to study.

## CubeSat limitation

The NEO-6M is a ground/airborne learning receiver, not an orbital CubeSat GNSS
receiver. Its documented operational limits are below low-Earth-orbit altitude
and velocity. This stage teaches UART, GNSS messages, validity, and fault
detection, but this exact receiver must not be selected as flight hardware.
