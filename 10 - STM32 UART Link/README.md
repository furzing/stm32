# 10 - STM32 UART Link

Saved: 2026-08-04

Build status: all C sources compiled with zero warnings, and the ELF linked
successfully. Image summary: text 27,804 bytes, data 92 bytes, bss 2,828 bytes.
Hardware status: awaiting the two-board bench test.

## What this version proves

This stage keeps the working GPS subsystem and adds a separate bidirectional
UART link between two STM32 boards:

~~~text
GPS -> USART1 -> Node 1 application -> USART2 -> PC console
                                  |
                                  +-> USART3 <-> Node 2
~~~

Keeping each purpose on a different peripheral is an architectural decision:

- USART1 owns the GPS and remains at 9600 baud.
- USART2 owns the PC diagnostic console and remains at 38400 baud.
- USART3 owns inter-processor communication and runs at 115200 baud.

This isolation prevents one device's configuration from silently changing
another device's data path and makes faults easier to localize.

## Why start with UART before a six-node network

The eventual system needs up to six STM32 nodes, for which a CAN bus with one
transceiver per board is a much better physical network than point-to-point
UART. UART is still the correct first experiment because it removes most
variables. If PING/PONG fails, the fault must be in pin selection, wiring,
ground reference, UART configuration, reception logic, or protocol parsing.

Moving directly to CAN would add bit timing, transceivers, termination,
arbitration, filters, and identifiers before the team has proven a shared
communications contract.

## Hardware reasoning

The UART wires are crossed:

~~~text
Node 1 PB10 TX  -> Node 2 PB11 RX
Node 1 PB11 RX  <- Node 2 PB10 TX
Node 1 GND      <-> Node 2 GND
~~~

TX pins are outputs and RX pins are inputs. Connecting TX to TX would make two
outputs electrically contend and neither receiver would receive the message.

Ground is not merely a safety wire. A UART receiver decides whether a signal
is high or low by comparing it to its own ground. Without a common reference,
the same wire voltage can mean different logic levels to the two boards.

Each board stays powered from its own USB connection. Connecting their 5 V or
3.3 V rails could make regulators back-power one another or create unwanted
current paths. Only the signal wires and common ground are shared.

On Node 1, the official Nucleo connector mapping exposes PB10 as Arduino D6 or
ST Morpho CN10 pin 25, and PB11 as ST Morpho CN10 pin 18.

## Why 115200 8-N-1

`8-N-1` means one start bit, eight data bits, no parity, and one stop bit. One
payload byte therefore occupies ten wire bits. At 115200 baud, the theoretical
payload limit is about 11,520 bytes per second:

~~~text
115200 wire bits/s / 10 wire bits per byte = 11520 payload bytes/s
~~~

Our one PING and one PONG per second use a tiny fraction of that bandwidth.
The higher baud rate gives headroom for later sensor telemetry, but reliability
still matters more than peak speed. Long wires, weak ground connections, and
electrical noise can force a lower rate or a more robust physical layer.

## Protocol and why every field exists

Node 2 sends:

~~~text
PING,2,42\r\n
~~~

Node 1 replies:

~~~text
PONG,1,42\r\n
~~~

- `PING`/`PONG` states the message purpose.
- `1` or `2` identifies the sender, catching crossed or misconfigured nodes.
- `42` is a sequence number, which detects missing, duplicate, or reordered
  transactions.
- `\r\n` frames the message so a continuous byte stream can be separated into
  complete records.

A changing LED or an arbitrary byte only proves that something happened. A
matched request and response sequence proves both wire directions, both UART
configurations, framing, parsing, and application execution.

## Code architecture

`LINK_StartReception()` clears state and arms the first one-byte USART3
interrupt. Every completed byte calls `HAL_UART_RxCpltCallback()`.

The callback performs only bounded work:

1. Ignore carriage return.
2. Append ordinary bytes to `building_line`.
3. On newline, publish a complete line to `ready_line`.
4. Rearm the next one-byte interrupt.

The main loop calls `LINK_TakeCompletedLine()` and `LINK_ProcessLine()`.
Parsing and transmission happen in main rather than inside the interrupt.
That design keeps interrupt latency predictable while USART1 is also receiving
GPS bytes.

The single ready buffer is sufficient for the one-message-per-second test. If
messages later arrive faster than main consumes them, `dropped_lines` exposes
the overload instead of silently overwriting unprocessed data. A ring buffer or
DMA becomes justified only when measurement proves this design insufficient.

## Diagnostics as engineering evidence

Every five seconds Node 1 prints:

~~~text
LINK STATUS: ONLINE pings=25 last_seq=24 age=7 ms lines=25 dropped=0 overflow=0 malformed=0 wrong_node=0 seq_gaps=0 uart_errors=0 tx_errors=0
~~~

Interpret the counters by layer:

| Observation | Likely layer | Next check |
|---|---|---|
| WAITING and lines=0 | Electrical/configuration | Common GND, crossed TX/RX, baud, correct pins |
| lines increase, malformed increases | Protocol | Exact commas, decimal fields, CRLF |
| wrong_node increases | Addressing | Node 2 must transmit source ID 2 |
| seq_gaps increase | Timing/data loss | Node-2 counter logic, wiring, CPU load |
| overflow increases | Framing/length | Missing newline or message over 63 characters |
| uart_errors increase | Electrical/UART | Baud mismatch, noise, loose ground, overrun |
| tx_errors increase | HAL/peripheral state | USART3 initialization and concurrent TX use |
| ONLINE becomes STALE | Liveness | Node 2 stopped or its TX path failed |

This is the engineering habit to build: do not ask only whether it works.
Design the software to reveal which layer failed when it does not work.

## Test procedure and acceptance criteria

1. Flash both boards separately and verify their local consoles.
2. Power down before wiring the boards together.
3. Connect common GND and cross USART3 TX/RX.
4. Power both boards independently over USB.
5. Observe Node 1 WAITING, then start Node 2 PING transmission.
6. Run at least 100 consecutive exchanges.
7. Remove Node-2 TX: Node 1 must become STALE.
8. Restore Node-2 TX: Node 1 must recover without reset.
9. Remove Node-1 TX: Node 2 must time out while Node 1 still counts PINGs.

The stage passes when 100 consecutive sequences match, all error counters stay
zero, each deliberately removed wire causes the predicted diagnostic, and the
link recovers after reconnection.

## Deliberately not implemented yet

- Sensor payload transfer.
- Binary message framing.
- Payload length and version fields.
- CRC protection.
- Retries or acknowledgements beyond PING/PONG.
- CAN transceivers, termination, identifiers, or bus arbitration.

Those are later stages. Adding them now would make a failure harder to locate
and would weaken, rather than strengthen, the engineering process.
