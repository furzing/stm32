# STM32 UART Link Contract - Test 10

Send this file to the Node-2 developer. We do not need to share projects as
long as both implementations obey this exact electrical and message contract.

## Roles

- Node 1: NUCLEO-F334R8, responder, local node ID `1`.
- Node 2: STM32F407 DevEBox, coordinator, local node ID `2`.

## Electrical contract

Both boards use 3.3 V UART logic and are powered independently by USB.

| Node 1 NUCLEO-F334R8 | Connects to | Node 2 STM32F407 |
|---|---|---|
| PB10 / USART3_TX | -> | PB11 / USART3_RX |
| PB11 / USART3_RX | <- | PB10 / USART3_TX |
| GND | <-> | GND |

Do not connect either board's 5 V or 3.3 V power rail to the other board.
The common ground is mandatory because UART voltages are measured relative to
ground. Cross TX to RX because outputs must drive inputs, never outputs.

On the NUCLEO-F334R8, PB10 is available as Arduino D6 and ST Morpho CN10 pin
25. PB11 is ST Morpho CN10 pin 18.

## UART configuration on both boards

| Setting | Required value |
|---|---|
| Peripheral | USART3 |
| Mode | Asynchronous |
| Baud rate | 115200 |
| Word length | 8 bits |
| Parity | None |
| Stop bits | 1 |
| Hardware flow control | None |
| Direction | Transmit and receive |
| USART3 global interrupt | Enabled |
| Line ending | CRLF (`\r\n`) |

## Test-10 message contract

Node 2 is the coordinator. Once per second it sends:

~~~text
PING,2,<sequence>\r\n
~~~

Node 1 validates the message and replies immediately:

~~~text
PONG,1,<same-sequence>\r\n
~~~

Example exchange:

~~~text
Node 2 -> Node 1: PING,2,42\r\n
Node 1 -> Node 2: PONG,1,42\r\n
~~~

`sequence` is an unsigned 32-bit decimal counter. Node 2 starts at zero and
increments after every transmitted PING. Node 2 must only accept a PONG when:

1. The verb is exactly `PONG`.
2. The source ID is exactly `1`.
3. The returned sequence equals the outstanding PING sequence.
4. The line terminates with CRLF.

Node 2 should report transmitted PINGs, valid PONGs, timeouts, malformed
messages, and round-trip time on its own debug console. A timeout of 500 ms is
reasonable for this one-message-per-second bench test.

## Bring-up order

1. Power and flash each board separately.
2. Confirm each board's own debug console still works.
3. Disconnect power before adding inter-board wires.
4. Connect GND only, then TX/RX crossed as shown above.
5. Power both boards by their own USB cables.
6. Confirm Node 1 prints `LINK STATUS: WAITING` before Node 2 starts.
7. Start Node 2 and confirm consecutive PING/PONG sequence numbers.
8. Disconnect one UART wire and confirm timeouts/errors are detected.
9. Reconnect it and confirm communication recovers.

## Acceptance criteria

- At least 100 consecutive PING/PONG exchanges succeed.
- Both boards agree on every returned sequence number.
- No malformed, wrong-node, overflow, or UART error counters increase.
- Removing Node-2 TX makes Node 1 change from ONLINE to STALE.
- Removing Node-1 TX makes Node 2 time out while Node 1 still receives PINGs.
- Reconnection restores operation without resetting either board.

This is intentionally a simple ASCII transport test. Sensor messages, binary
framing, CRC, arbitration, and the future six-node CAN network come only after
this electrical and bidirectional foundation passes.
