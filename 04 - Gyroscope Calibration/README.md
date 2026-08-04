# 04 - Gyroscope Calibration

Saved: 2026-07-30

## Hardware connections

- Board: NUCLEO-F334R8
- Sensor: ICM-20948 using I2C1
- PB8: I2C clock
- PB9: I2C data
- AD0: connected to ground, selecting address `0x68`
- Serial output: ST-LINK Virtual COM Port at 38400 baud

## What this program does

- Finds the sensor on the I2C bus.
- Confirms the sensor identity is `0xEA`.
- Resets and wakes the sensor.
- Configures the accelerometer for +/-2 g.
- Configures the gyroscope for +/-250 degrees per second.
- Reads acceleration, rotation, and temperature together.
- Prints converted measurements every 200 ms.
- Collects 500 stationary gyro readings during startup.
- Calculates and subtracts the X, Y, and Z gyro offsets.
- Blinks the Nucleo user LED as a heartbeat.

## Startup requirement

Keep the sensor completely motionless while the five-second gyroscope
calibration message is displayed.

## How the hardware data reaches the terminal

1. `I2C_Scan()` checks each legal 7-bit I2C address. AD0 is grounded, so this
   board should answer at `0x68`.
2. `ICM20948_CheckIdentity()` reads `WHO_AM_I`. Receiving `0xEA` proves that
   the responding device is an ICM-20948, not merely something using the same
   bus address.
3. `ICM20948_Initialize()` resets the device, wakes it, enables all axes,
   selects the +/-2 g and +/-250 dps ranges, and configures its sample rate.
4. `ICM20948_ReadRawData()` performs one burst read so acceleration,
   gyroscope, and temperature belong to approximately the same instant.
5. `ICM20948_CalibrateGyroscope()` averages 500 stationary readings and stores
   the zero-rate error for each axis.
6. `ICM20948_PrintSample()` subtracts those errors, converts raw counts into
   physical units, and sends text through USART2.
7. `main()` repeats the read/print process and toggles LD2 as a heartbeat.

## Why gyroscope calibration is necessary

A stationary MEMS gyroscope rarely returns exactly zero. For example, an
average raw X value of 65 counts at 131 counts per degree/second represents:

```text
65 / 131 = 0.496 degrees/second
```

If that bias were integrated for one minute, the calculated angle could drift
by nearly 30 degrees even though the sensor never moved. The program estimates
the bias using:

```text
bias = sum of stationary samples / number of samples
corrected rate = raw rate - bias
```

The calibration is only valid if the IMU is motionless. Movement becomes part
of the average and is then incorrectly subtracted from later readings.

## C concepts in this version

### Fixed-width integer types

`int16_t` matches the sensor's signed 16-bit output registers. `int32_t` stores
the calculated bias, while `int64_t` safely accumulates hundreds of samples
without overflowing.

### Structures

`ICM20948_RawData` groups values from one measurement. The
`ICM20948_GyroCalibration` structure keeps the three bias values together.
This is clearer and safer than maintaining many unrelated global variables.

### Pointers

A parameter such as:

```c
ICM20948_GyroCalibration *calibration
```

is the address of the caller's structure. Writing
`calibration->bias_x_raw` changes the original object. The function first
checks the pointer against `NULL` so an invalid address is not dereferenced.

### Bit shifting and byte order

Each measurement arrives as a high byte and low byte. This expression rebuilds
one signed value:

```c
(int16_t)(((uint16_t)high_byte << 8) | low_byte)
```

The shift moves the high byte into bits 15:8; bitwise OR adds the low byte.
The final cast interprets the combined bits as a signed two's-complement value.

### Return-status checking

Hardware functions return `HAL_OK`, `HAL_ERROR`, or another HAL status. Every
important transaction is checked. Continuing after a failed register write
could make later measurements look believable while using the wrong sensor
configuration.

### Integer formatting

Float formatting is often disabled in small embedded C libraries to save flash.
`FormatMilliValue()` therefore stores values in thousandths: `-1234` becomes
`-1.234`. This provides readable decimals without enabling heavy `%f` support.

## What to verify

- `WHO_AM_I` must equal `0xEA`.
- The calibration must complete without an I2C error.
- Corrected gyro values should fluctuate near zero while stationary.
- Each gyro axis should respond strongly when rotating around its matching
  sensor axis.
- Acceleration magnitude should be near 1 g while stationary, although its
  distribution among X, Y, and Z depends on orientation.
