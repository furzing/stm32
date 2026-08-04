# 09 - GPS Position Parsing

Saved: 2026-08-04

Build status: clean build with zero compiler errors and zero warnings.

Hardware status: stage 08 proved live NMEA reception indoors. Stage 09 still
needs to be flashed and validated first with the indoor no-fix data, then
outdoors with a real navigation fix.

## Requirement

Turn received NMEA text into a position that downstream firmware can trust.

The implementation must never present corrupted, invalid, or stale coordinates
as a current valid position. This is more important than merely extracting two
numbers from a comma-separated string.

## Engineering decision sequence

The processing order is deliberate:

1. Receive a complete line without interpreting it.
2. Verify the NMEA checksum.
3. Identify the sentence type.
4. Preserve empty fields while splitting.
5. Check the sentence-specific validity indicator.
6. Convert latitude and longitude.
7. Timestamp the accepted fix.
8. Report validity, age, and diagnostic counters.

Changing this order creates unsafe failure modes. For example, parsing
coordinates before checksum verification can turn a corrupted byte into a
plausible but false location.

## Data path

~~~text
GPS UART byte
    |
USART1 interrupt
    |
building_line[]
    |
complete NMEA sentence
    |
main loop
    |
checksum verification
    |
GGA/RMC validity checks
    |
coordinate conversion
    |
GPS_Navigation state
    |
USART2 terminal output
~~~

The interrupt remains responsible only for reliable acquisition and framing.
Parsing stays in the main loop because it is slower and is not time-critical.

## NMEA checksum

A sentence has this shape:

~~~text
$GPRMC,...*6A
~~~

The receiver XORs every byte after the dollar sign and before the asterisk.
The result is compared with the two hexadecimal characters after the
asterisk.

~~~text
calculated checksum == transmitted checksum -> sentence may be parsed
calculated checksum != transmitted checksum -> reject whole sentence
missing/invalid checksum syntax             -> malformed sentence
~~~

Mismatch and malformed counters are separate because they suggest different
faults:

- Repeated mismatches suggest transport noise, baud errors, or damaged bytes.
- Malformed lines suggest framing, truncation, unexpected protocol format, or
  parser assumptions that need review.

## Why empty fields must be preserved

The indoor no-fix line contains repeated commas:

~~~text
$GPGGA,,,,,,0,00,99.99,,,,,,*48
~~~

Each empty position still has a defined field number. Standard strtok-style
tokenization collapses repeated delimiters, shifting later fields into the
wrong positions. GPS_SplitFields replaces commas one at a time and records the
start of every field, including empty ones.

## Sentence selection

Only GGA and RMC are interpreted in this stage.

GGA supplies:

- Latitude and longitude.
- Fix quality.
- Number of satellites used.

RMC supplies:

- Latitude and longitude.
- A/V navigation validity.

GSA, GSV, GLL, and VTG still pass checksum accounting but are ignored by the
position parser. Implementing only the data required now keeps the parser
small enough to verify.

The final three characters of the header identify the type, allowing both
GPS-only and multi-GNSS talker prefixes:

~~~text
$GPGGA -> GGA
$GNGGA -> GGA
~~~

## Coordinate mathematics

NMEA latitude is ddmm.mmmm:

~~~text
4807.038,N
= 48 degrees + 7.038 minutes / 60
= 48.1173000 degrees
~~~

NMEA longitude is dddmm.mmmm:

~~~text
01131.000,E
= 11 degrees + 31.000 minutes / 60
= 11.5166667 degrees
~~~

South and west are negative:

~~~text
N and E -> positive
S and W -> negative
~~~

The parser rejects:

- Minutes greater than or equal to 60.
- Latitude beyond 90 degrees.
- Longitude beyond 180 degrees.
- A latitude with E/W or longitude with N/S.
- Missing, nonnumeric, or structurally wrong coordinate fields.

## Why fixed-point is used

Coordinates are stored as signed degrees multiplied by 10^7:

~~~text
48.1173000 degrees -> 481173000
-35.9123456 degrees -> -359123456
~~~

This decision gives:

- A fixed and documented resolution.
- Deterministic integer behavior.
- Straightforward future CAN serialization.
- No dependency on floating-point printf configuration.
- The same stored representation on different STM32 families.

The numerical resolution is finer than the physical accuracy of the NEO-6M.
Extra digits represent the chosen storage scale, not guaranteed positioning
accuracy.

## Validity and freshness are different

Validity answers:

~~~text
Did the GPS declare this sentence usable when it was produced?
~~~

Freshness answers:

~~~text
How long ago was the last accepted position produced?
~~~

The code records last_fix_ms and defines a 2000 ms freshness threshold.
A once-valid coordinate must not remain trusted forever after the antenna is
disconnected or UART traffic stops.

The current terminal status uses:

- VALID: latest sentence says valid and the fix is recent.
- NO_FIX: GGA quality is zero or RMC status is V.
- STALE: the last accepted fix is older than the freshness threshold.

Future control or network code must check both validity and age before using a
position.

## Indoor expected behavior

The existing indoor data has valid NMEA syntax but reports no navigation fix.
Expected output is periodic status rather than coordinates:

~~~text
GPS STATUS: ... checksum_ok=... checksum_bad=0 malformed=0 fixes=0 state=NO_FIX sats=0 ...
~~~

This proves the parser correctly distinguishes a valid message from a valid
position.

## Outdoor expected behavior

After the receiver obtains a valid GGA or RMC fix:

~~~text
GPS FIX: latitude=31.xxxxxxx deg longitude=35.xxxxxxx deg source=GGA quality=1 satellites=8 age=0 ms
~~~

GGA and RMC may each update the position, so two position lines per GPS
measurement cycle are acceptable in this teaching stage.

## Diagnostics and decisions

| Observation | Likely layer | Next decision |
|---|---|---|
| bytes=0 | Power, wiring, or USART | Check VCC, GND, TX to PA10 |
| UART errors increase | Electrical/baud | Check ground, baud, noise, loose wiring |
| lines increase, checksum_bad increases | Data integrity | Inspect UART waveform and connections |
| checksum_ok increases, fixes=0 | RF/navigation | Test outdoors and inspect antenna |
| malformed increases with checksum_ok | Parser/protocol | Capture the exact rejected sentence |
| fixes increase, state=VALID | Full navigation chain | Compare against a known map position |
| state becomes STALE | Update stopped | Reject position and diagnose RF/UART |

## Acceptance criteria

Stage 09 is complete only when:

1. Indoor no-fix sentences increase checksum_ok without producing a false fix.
2. checksum_bad and malformed remain zero with stable wiring.
3. An outdoor GGA/RMC fix produces sensible signed decimal coordinates.
4. The position agrees with a trusted map location within normal receiver
   accuracy.
5. Disconnecting or shielding the antenna causes the state to become NO_FIX or
   STALE rather than continuing to claim a current valid fix.

## Not implemented yet

- UTC time/date parsing.
- Altitude parsing.
- Speed and course parsing.
- GSA/GSV satellite diagnostics.
- A ring buffer or DMA receiver.
- Binary UBX protocol.
- CAN serialization.
- Flight-qualified GNSS hardware behavior.

These should be separate verified stages rather than being added before the
position foundation is proven.
