# Changelog

## 0.1.0

The first release: a driver for TDK InvenSense's MPU-6050 six-axis motion
tracking device on an I2C bus, from the product specification
PS-MPU-6000A-00 revision 3.4 and the register map RM-MPU-6000A-00
revision 4.0.

- `Mpu6050` (I2C), the type a board's device table builds from the part's
  wiring, implementing devices-nv's `Accelerometer`, `Gyroscope`, `Imu`,
  `Thermometer` and `Identifiable`.
- The bring-up: the device reset with its 100 ms wait and the poll of
  PWR_MGMT_1 for 0x40, WHO_AM_I (only 0x68), the wake on the PLL with the X
  gyroscope as the clock reference, the digital low-pass filter at 44 Hz,
  100 Hz, ±2 g and ±250 dps, and the data-ready flag.
- The accelerometer in milli-g and the gyroscope in milli-degrees per
  second at the sensitivities of the product specification, the ranges, the
  one sample rate for both sensors from the output rate of the filter in use
  and `SMPLRT_DIV` with a rate of 0 as standby of one sensor's axes, the
  digital low-pass filter, the paired sample from one burst of fourteen
  registers, the data-ready flag, and the temperature as `TEMP_OUT / 340 +
  36.53`.
- The power states (sleep, run, and the accelerometer-only cycle mode with
  its wake-up rate), the clock source and the temperature sensor's switch.
- The interrupt pin's route, polarity and output type, keeping the other
  bits of INT_PIN_CFG.
- The FIFO with 12-byte and 6-byte samples, the count in samples, and an
  overflow detected from the count of 1024 and emptied.
- The accelerometer's and the gyroscope's self-tests as the register map
  defines them, the factory trims computed from the SELF_TEST registers
  with the register map's formulas, and the check against ±14 %.
- The device program `examples/imu_reader.nv`, a reading loop. Built for
  the nRF52840 DK it is 15,832 B of flash and for the nRF54LM20 DK 17,296 B.
  `tests/device_build.sh` builds it for both and runs the emulated board's
  image.
- Not verified on hardware: no board on the bench carries the chip. A reply
  from an MPU-6050 over the Nordic boards' TWIM buses is NOT VERIFIED until
  the driver rig.
- Not implemented: the Digital Motion Processor, the auxiliary I2C master,
  the motion-detection interrupt, the frame sync pin, the INT pin's latch
  modes, and the async form of the traits.

### Known

- The gyroscope trim formula is the register map's, and the limit of ±14 %
  is the product specification's for revision D parts and later.
- The documents do not say how a burst read behaves at FIFO_R_W. The
  driver reads one sample per burst.
