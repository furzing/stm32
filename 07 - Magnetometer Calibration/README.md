# 07 - Magnetometer Calibration

Saved: 2026-07-30

Hardware validation status: complete. The guided 30-second calibration,
coverage checks, bounded I2C retries, hard-iron correction, soft-iron scaling,
and corrected magnetic output were confirmed on hardware.

## Goal of this version

Raw magnetometer readings are not ready for heading calculations. The sensor
measures Earth's field plus magnetic errors from the PCB, connectors, wires,
nearby steel, permanent magnets, and current-carrying conductors.

This version learns two first-order corrections:

- **Hard-iron offset:** a constant X/Y/Z field added by magnetized objects.
- **Diagonal soft-iron scale:** unequal stretching of the three axes caused by
  nearby magnetic material.

The result is suitable for learning and initial heading tests. A later
CubeSat-grade calibration should use a full 3x3 ellipsoid correction matrix,
temperature characterization, and calibration in the assembled spacecraft.

## Connections

| Signal | ICM-20948 board | NUCLEO-F334R8 |
|---|---|---|
| Power | VCC | 3V3 |
| Ground | GND | GND |
| I2C clock | SCL | PB8 / Arduino D15 |
| I2C data | SDA | PB9 / Arduino D14 |
| Address select | AD0 | GND |

Grounding AD0 selects the ICM-20948 address `0x68`. The AK09916 magnetometer
is inside the ICM-20948 package at auxiliary address `0x0C`; it is not directly
connected to the STM32 I2C pins.

## Test procedure

1. Reset or flash the board.
2. Keep the IMU completely still during the five-second gyroscope calibration.
3. The terminal gives a five-second warning before magnetic calibration.
4. Move the board away from the laptop, tools, speakers, magnets, and steel.
5. For 30 seconds, rotate it slowly through every possible 3D orientation:
   point every face upward and downward, rotate around all three axes, and make
   slow figure-eight motions.
6. Do not translate it close to magnetic objects while rotating it.
7. Read the reported axis spans, offsets, scale factors, and corrected field.

The calibration fails deliberately if fewer than 200 distinct samples are
collected or any axis spans less than 30 microtesla. That usually means the
sensor was not rotated far enough around that axis.

## Calibration mathematics

For each raw magnetic axis, the program records a minimum and maximum:

```text
offset = (maximum + minimum) / 2
radius = (maximum - minimum) / 2
```

Subtracting the midpoint recenters the magnetic measurements and removes the
estimated hard-iron error:

```text
centered = raw - offset
```

The ideal 3D rotation produces the same radius on all axes. This version
computes their average radius and scales each axis toward it:

```text
average_radius = (radius_x + radius_y + radius_z) / 3
scale_x = average_radius / radius_x
corrected_x = (raw_x - offset_x) * scale_x
```

The same operation is performed for Y and Z. Finally, raw LSB values are
converted using the AK09916 sensitivity of 0.15 microtesla per LSB.

## How the code is organized

- `ICM20948_RawData` holds one hardware sample exactly as signed register
  values. Keeping raw data separate makes register and byte-order bugs visible.
- `ICM20948_GyroCalibration` stores startup zero-rate gyro bias.
- `ICM20948_Attitude` stores the state of the roll/pitch complementary filter.
- `ICM20948_MagnetometerData` caches the latest distinct magnetic vector.
- `ICM20948_MagnetometerCalibration` stores offsets, scale factors, sample
  count, and a `valid` flag.
- `ICM20948_CalibrateMagnetometer()` runs the guided 30-second collection,
  retries transient I2C errors, checks coverage, and calculates the calibration.
- `ICM20948_ApplyMagnetometerCalibration()` performs offset subtraction,
  scaling, and conversion to microtesla.
- `ICM20948_PrintSample()` prints raw magnetic data, corrected data, and the
  corrected vector magnitude.
- `main()` performs initialization, calibrations, and then runs the continuous
  100 Hz attitude loop with 5 Hz terminal output.

## C concepts introduced

### Structures

A `struct` groups values that belong together. Passing a pointer such as
`ICM20948_MagnetometerCalibration *calibration` lets a function update the
original calibration object rather than a temporary copy.

### Minimum and maximum tracking

The extrema start at `INT16_MAX` and `INT16_MIN`. Every new measurement is
compared with the stored limits:

```c
if (sample.mag_x < minimum_x) minimum_x = sample.mag_x;
if (sample.mag_x > maximum_x) maximum_x = sample.mag_x;
```

### Integer and floating-point roles

Register samples and extrema remain integers because the hardware produces
signed 16-bit values. Calibration offsets and scales use `float` because
midpoints may contain half an LSB and scale factors are fractional.

### Defensive status checking

Functions return `HAL_OK` or an error. The caller checks each result, so an I2C
failure or insufficient calibration cannot silently produce trusted heading
data.

The calibration distinguishes a transient error from a lost connection. It
counts every failed read, retries isolated failures, and aborts after 20
consecutive failures. At a 10 ms retry interval, that represents approximately
200 ms of continuous communication loss:

```c
if (consecutive_i2c_failures >=
    ICM20948_MAG_CALIBRATION_MAX_CONSECUTIVE_I2C_FAILURES)
{
    return HAL_ERROR;
}
```

This is bounded fault handling: the software is tolerant, but it does not retry
forever and pretend that a broken electrical connection is healthy.

### Timer arithmetic

Elapsed time is calculated with unsigned subtraction:

```c
(uint32_t)(HAL_GetTick() - start_ms)
```

This remains correct even when the millisecond counter eventually wraps.

## Expected terminal output

During calibration:

```text
MAG CAL: 25 s left, ... samples, ... I2C retries.
Captured spans [uT]: X=... Y=... Z=... | samples=... retries=...
Hard-iron offset [uT]: X=... Y=... Z=...
Soft-iron scale:       X=... Y=... Z=...
Magnetometer calibration complete.
```

During normal operation:

```text
MAG_RAW[uT] X=... Y=... Z=... | unchanged=... ms
MAG_CAL[uT] X=... Y=... Z=... | |B|=... uT
```

`unchanged` means time since the vector value last changed. It is not a sensor
disconnection timer. When the IMU is perfectly still, repeated identical
samples may legitimately make this number grow.

## I2C retry interpretation

- `0 retries`: ideal bench connection during the entire rotation.
- A small nonzero count: the calibration can complete, but inspect wire strain,
  jumper fit, solder joints, and grounding.
- A rapidly increasing count: stop trusting the test and fix the hardware.
- `20 consecutive I2C failures`: the firmware aborts calibration and reports
  the connection as lost.

Retry logic improves fault tolerance; it does not make intermittent wiring
acceptable for a CubeSat module.

## How to judge the result

- All three captured spans should be substantial; one much smaller span means
  that axis was not fully rotated.
- Scale values should be positive and reasonably close to one. A very large or
  small value suggests poor coverage or strong nearby magnetic material.
- After calibration, the magnitude `|B|` should remain much more consistent as
  the sensor rotates in one location.
- Moving the sensor near steel or electric currents can still change `|B|`;
  calibration cannot remove a magnetic disturbance that changes over time.

## First successful hardware result

The first completed 30-second calibration produced:

```text
Captured spans [uT]: X=+83.100 Y=+68.550 Z=+66.600
Samples: 492
I2C retries: 4
Hard-iron offset [uT]: X=-33.900 Y=+1.425 Z=+22.350
Soft-iron scale:       X=+0.875 Y=+1.061 Z=+1.092
```

Interpretation:

- All axes exceeded the 30 microtesla coverage requirement.
- X covered a larger radius than Y and Z, so its scale factor is below one.
- Y and Z covered smaller radii, so their scale factors are above one.
- The large X and Z offsets show substantial constant magnetic bias in the
  tested assembly/environment.
- Four isolated I2C retries were recovered successfully. This is acceptable
  for proving the retry logic, but the wiring still requires inspection and
  strain relief before it can be considered reliable hardware.
- The displayed corrected magnitude in the captured stationary output was
  approximately 27 to 30.5 microtesla. A dedicated slow-rotation validation
  should measure its full minimum and maximum before heading is trusted.

## Current limitation

The calculated values are stored only in RAM. Resetting or power-cycling the
board runs the calibration again. This is intentional for the first test.
After the numbers are validated, a later version can store them in flash or
load approved constants at startup.
