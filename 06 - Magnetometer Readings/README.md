# 06 - Magnetometer Readings

Saved: 2026-07-30

Hardware validation status: complete. AK09916 identity, numeric X/Y/Z changes,
overflow handling, and delayed auxiliary reads were confirmed on hardware.

## What this program adds

- Enables the ICM-20948 internal auxiliary I2C controller.
- Uses the datasheet-recommended auxiliary bus clock setting.
- Resets the internal AK09916 magnetometer.
- Checks for the expected AK09916 identity value `0x09`.
- Selects continuous magnetometer measurement at 50 Hz. This avoids the
  data-ready aliasing observed when its previous 100 Hz rate nearly matched
  the approximately 102.3 Hz ICM-20948 auxiliary-controller rate.
- Reads the 50 Hz magnetometer through auxiliary Slave 0 approximately
  20.5 times per second. The read rate is deliberately below the measurement
  rate so every auxiliary transaction finds completed magnetic data.
- Copies magnetometer data into the ICM-20948 external-sensor registers.
- Reads magnetic X, Y, and Z together with the existing motion data.
- Handles the magnetometer's little-endian measurement format.
- Reads the required final status register.
- Rejects magnetic samples marked as overflowed.
- Caches every valid 50 Hz magnetic sample so the 5 Hz terminal output
  cannot miss it between print intervals.
- Accepts each distinct external-sensor shadow vector only once, preventing
  repeated reads of the same shadow block from falsely resetting its age.
- Reports the cached sample age and detects data that has become stale.
- Prints raw status bytes and the overflow count if no valid sample has
  arrived, making the next hardware or configuration fault easy to identify.
- Converts magnetic measurements to microtesla.
- Keeps the existing gyro calibration and roll/pitch filter.

## Expected startup output

```text
AK09916 identity returned 0x09 (expected 0x09).
AK09916: 50 Hz measurement, approximately 20 Hz reads.
```

## Expected measurement output

```text
MAG[uT] X=... Y=... Z=... | age=... ms
```

The values should change when the sensor is rotated relative to Earth's
magnetic field or moved near magnetic material. These readings are not yet
calibrated and must not be used for heading until hard-iron and soft-iron
calibration is added.

If the firmware has never received a valid magnetic sample, it prints:

```text
MAG waiting: ST1=0x.. ST2=0x.. raw=(...,...,...) overflows=...
```

If samples stop updating for more than 500 ms after valid data was received,
it reports the last sample as stale instead of silently reusing it.

## Important lesson from the `age` field

In this archived version, `age` is actually time since the cached X/Y/Z vector
last **changed**, because identical external-shadow vectors are accepted only
once. A stationary sensor can therefore show a large age even when communication
is healthy. The low ages seen while moving proved that new samples arrived.

Calling unchanged data “stale” is too strong: an identical magnetic vector can
be physically legitimate. Stage 07 corrects the terminal label to `unchanged`
and does not treat value equality alone as a disconnection.

## The nested I2C architecture

The STM32 communicates with the ICM-20948 at address `0x68` using PB8 and PB9.
The AK09916 magnetometer is internal to the IMU package on a separate auxiliary
bus at address `0x0C`. The STM32 cannot scan that internal address directly.

The data path is:

```text
STM32 I2C1
    -> ICM-20948 main I2C interface
        -> ICM-20948 auxiliary I2C controller
            -> AK09916 magnetometer
        -> EXT_SENS_DATA shadow registers
    -> STM32 burst read
```

This explains why detecting `0x68` proves only the main IMU connection. The
separate AK09916 identity value `0x09` proves the internal path.

## Slave 4 versus Slave 0

- **Auxiliary Slave 4** performs one register transaction at a time. Functions
  `AK09916_WriteRegister()` and `AK09916_ReadRegister()` use it for reset,
  identity checking, and selecting the measurement mode.
- **Auxiliary Slave 0** repeatedly reads the nine-byte block from `ST1` through
  `ST2`. The ICM-20948 then exposes that block in its external-sensor shadow
  registers for the STM32.

`ICM20948_WaitForSlave4()` polls completion and checks the NACK flag. A NACK
means the internal magnetometer did not acknowledge an auxiliary transaction.

## Why ST1 and ST2 are included

`ST1.DRDY` indicates that a new magnetic conversion is ready. `ST2.HOFL`
indicates magnetic overflow; those values must be rejected. Reading through
`ST2` also completes the AK09916 measurement-read protocol.

The complete block is read even though only six bytes contain X/Y/Z. Status
bytes are part of deciding whether measurement bytes are trustworthy.

## Byte order lesson

ICM-20948 accelerometer and gyro words arrive high byte first. AK09916 magnetic
words arrive low byte first. The reconstruction expressions are therefore
different:

```c
/* Main motion sensor: big-endian register order. */
value = (int16_t)(((uint16_t)high << 8) | low);

/* Magnetometer: little-endian register order. */
value = (int16_t)(((uint16_t)second_byte << 8) | first_byte);
```

Using the wrong order can still produce changing numbers, which is why
“numbers are appearing” is not sufficient validation.

## The sample-rate problem we discovered

The gyro-enabled auxiliary controller operated near 102.3 Hz while the first
magnetometer attempt used 100 Hz. Auxiliary reads consume the AK09916 ready
flag, so the nearly equal clocks produced a beat pattern: the STM32 only
observed a ready shadow about every 440 ms.

The final stage-06 configuration:

- Produces magnetic conversions at 50 Hz.
- Executes automatic Slave-0 transfers at approximately 20.5 Hz.
- Keeps the 100 Hz accel/gyro attitude loop unchanged.
- Caches distinct magnetic vectors independently of the 5 Hz UART output.

This is an embedded-systems timing lesson: two components can both be “fast”
while their relative phase makes valid events almost invisible.

## C concepts in this version

### Bit masks

Status bits are tested with bitwise AND:

```c
if ((status_2 & AK09916_ST2_OVERFLOW) != 0U)
```

The mask isolates one bit without changing the other status bits.

### Register constants

Named `#define` values replace unexplained hexadecimal numbers. For example,
`AK09916_REG_CNTL2` communicates purpose better than scattering `0x31`
throughout the program.

### Layered functions

Low-level bank selection and I2C reads are separate from AK09916 transactions,
sensor initialization, caching, attitude estimation, and terminal formatting.
This layering reduces the amount of code that must be inspected when one layer
fails.

### Caching between rates

The sensor path, attitude estimator, and UART output run at different rates.
`ICM20948_MagnetometerData` preserves a valid magnetic sample so a slower print
operation does not have to occur during the brief hardware-ready event.

## What was verified

- Main device at `0x68` with `WHO_AM_I = 0xEA`.
- Internal AK09916 with identity `0x09`.
- Magnetic values changed with physical rotation.
- Values remained stable when the sensor was held still.
- Overflowed samples were rejected.
- Accel/gyro and roll/pitch processing continued alongside magnetometer reads.

These values remain uncalibrated in stage 06 and must not yet be interpreted as
north or used for yaw.
