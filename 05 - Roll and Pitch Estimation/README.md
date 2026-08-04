# 05 - Roll and Pitch Estimation

Saved: 2026-07-30

## What this program adds

- Keeps the five-second stationary gyroscope calibration.
- Reads the accelerometer and gyroscope approximately every 10 ms.
- Calculates tilt from the gravity vector.
- Integrates gyroscope X and Y rotation using the measured loop time.
- Uses a complementary filter to combine both measurements.
- Produces roll about the sensor X axis.
- Produces pitch about the sensor Y axis.
- Prints attitude and sensor measurements every 200 ms.
- Recovers from a long sensor-data interruption without integrating a false
  angular jump.

## Filter behavior

The accelerometer prevents long-term roll and pitch drift by providing a
gravity reference. The gyroscope provides smooth response during fast motion.

This program does not calculate yaw. Gravity cannot provide an absolute
heading reference, so yaw will be added after the magnetometer is working and
calibrated.

## Startup requirement

Keep the sensor completely motionless during the gyroscope calibration. After
the calibration completes, move the board slowly through known orientations
and verify the roll and pitch signs and axes.

## Coordinate meaning

- **Roll** is rotation about the sensor X axis.
- **Pitch** is rotation about the sensor Y axis.
- **Yaw** would be rotation about Z, but gravity alone cannot determine which
  horizontal direction is north.

The signs depend on the ICM-20948 axis markings and the equations used in this
program. Before installing the sensor in a spacecraft module, the sensor frame
must be mapped explicitly to the spacecraft body frame.

## How the estimator works

`ICM20948_UpdateAttitude()` receives one synchronized raw sample, the measured
gyro biases, elapsed time `dt`, and a pointer to the persistent attitude state.

First it converts acceleration into g and corrected angular rate into degrees
per second. Gravity-based tilt is then calculated with `atan2f()`:

```text
accelerometer roll  = atan2(ay, az)
accelerometer pitch = atan2(-ax, sqrt(ay^2 + az^2))
```

The gyroscope predicts how the previous angle changed:

```text
predicted angle = previous angle + angular rate * dt
```

Finally, the complementary filter combines both estimates:

```text
angle = alpha * gyro prediction
      + (1 - alpha) * accelerometer tilt
```

For this version, `alpha` is calculated from a 0.5-second time constant instead
of being a magic fixed number:

```text
alpha = time_constant / (time_constant + dt)
```

At a 10 ms update period, alpha is approximately 0.980. The gyro therefore
controls rapid response, while gravity slowly removes long-term drift.

## Why neither sensor is sufficient alone

The gyroscope measures rotation rate smoothly, but integrating even a small
remaining bias creates angle drift. The accelerometer gives an absolute gravity
direction when stationary, but it also measures vibration and linear
acceleration. During spacecraft thrust or strong bench motion, acceleration is
not purely gravity, so accelerometer-only tilt becomes wrong.

The complementary filter is a simple form of sensor fusion: each sensor is
trusted where its physics is strongest.

## Function guide

- `ICM20948_CalibrateGyroscope()` measures stationary rate bias.
- `ICM20948_ReadRawData()` acquires one coherent motion sample.
- `ICM20948_UpdateAttitude()` performs conversion, tilt mathematics,
  integration, and complementary filtering.
- `ICM20948_PrintSample()` separates the fast estimator from the slower
  terminal display.
- `main()` schedules estimation approximately every 10 ms and output every
  200 ms.

## C concepts in this version

### Persistent state

`ICM20948_Attitude` stores roll, pitch, and an `initialized` flag. The structure
survives between calls because it is created in `main()` and passed by pointer.
An estimator cannot work if its previous angle is recreated as zero every loop.

### Floating-point calculations

Angles, time in seconds, square roots, and trigonometric results use `float`.
The STM32F334 Cortex-M4F contains a single-precision floating-point unit, so
these operations are reasonable for this learning estimator.

### Measured elapsed time

The code derives `delta_time_s` from `HAL_GetTick()` rather than assuming that
every loop is exactly 10 ms. Integration error is proportional to timing error,
so the actual elapsed time matters.

### Initialization and recovery

On the first sample, the estimator initializes from gravity. If a sensor
interruption makes `dt` exceed 100 ms, it also reinitializes instead of
integrating one huge false angular step. This is an example of defensive
state-machine behavior.

### Different processing rates

Estimation runs near 100 Hz for smooth dynamics; serial output runs at 5 Hz so
human-readable text does not dominate CPU time or UART bandwidth. Fast
processing and slow reporting are separate responsibilities.

## Bench tests and expected angles

- Place the board flat: roll and pitch should be near their reference values.
- Raise one side slowly: primarily one angle should change.
- Rotate quickly and stop: response should be smooth, then settle without
  continuous drift.
- Hold the board still for a minute: large continuing drift indicates poor
  gyro calibration or mechanical movement.
- Briefly translate the board without rotating it: the accelerometer may
  disturb tilt temporarily, demonstrating why fusion is necessary.

This is not yet a full spacecraft attitude solution. It has no yaw reference,
no quaternion representation, no vibration qualification, and no estimator
covariance or fault detection.
