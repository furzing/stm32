# Saved Code Versions

Each saved version uses a simple number followed by a clear description of
what the program does.

## Development stages

| Number | Name | What it does |
|---|---|---|
| 01 | I2C Scanner | Searches the I2C bus and prints responding addresses. |
| 02 | Sensor Identity Check | Confirms the ICM-20948 by reading `WHO_AM_I = 0xEA`. |
| 03 | Motion Readings | Prints acceleration, gyroscope, and temperature readings. |
| 04 | Gyroscope Calibration | Measures gyro offsets at startup and corrects the readings. |
| 05 | Roll and Pitch Estimation | Fuses accelerometer and gyro data into roll and pitch angles. |
| 06 | Magnetometer Readings | Reads the internal AK09916 magnetic field on three axes. |
| 07 | Magnetometer Calibration | Learns hard-iron offsets and per-axis soft-iron scales from a guided 3D rotation. |
| 08 | GPS NMEA Reception | Receives complete raw NMEA sentences from the NEO-6M using interrupt-driven USART1. |

Saving was requested after stage 04 was completed, so **04 - Gyroscope
Calibration** is the first exact saved project. Stages 01 through 03 are
listed to preserve the development history, but they are not exact source
archives.

## Contents of a saved version

- `Program Code - main.c.txt`: the main program code for quick inspection.
- `Pin Configuration - STM_Testing.ioc.txt`: the CubeMX pin and peripheral
  configuration.
- `README.md`: a lesson explaining the hardware path, mathematics, important C
  concepts, function organization, test procedure, limitations, and expected
  output for that exact version.
- A ZIP with the same clear name: the complete restorable CubeIDE project.
  Each ZIP also contains the same lesson as `CODE_EXPLANATION.md`.

The code snapshot ends in `.txt` so CubeIDE cannot accidentally compile a
second copy of `main()`.

## Restore a complete saved project

1. Extract the selected ZIP into a new directory.
2. In STM32CubeIDE, select **File > Import > Existing Projects into
   Workspace**.
3. Select the extracted project directory.
4. Build and flash normally.

Using a new directory prevents old and new generated project files from being
mixed together.
