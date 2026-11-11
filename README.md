# mpu6050-nv

A driver for the MPU-6050, a six-axis motion tracking device from TDK
InvenSense with a three-axis gyroscope, a three-axis accelerometer and a
temperature sensor, on an I2C bus, specified in "MPU-6000 and MPU-6050
Product Specification", document PS-MPU-6000A-00, revision 3.4, and "MPU-6000
and MPU-6050 Register Map and Descriptions", document RM-MPU-6000A-00,
revision 4.0
([invensense.tdk.com/products/6-axis/mpu-6050](https://www.invensense.tdk.com/en-us/products/motion-tracking/6-axis/mpu-6050)).
The driver implements the `Accelerometer`, `Gyroscope`, `Imu`,
`Thermometer` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv). It reads the
accelerometer in milli-g and the gyroscope in milli-degrees per second.
TDK lists the part as end of life; it has a large installed base on
breakout boards.

**Hardware: NOT VERIFIED.** No board on the bench carries an MPU-6050
yet. The device program below builds for the nRF52840 DK and the
nRF54LM20 DK, and the driver's functions run over a double of the part
that follows the register map and the product specification. A reading
from a part on a board has not been taken, and the program has not been
flashed.

## The device program

`examples/imu_reader.nv` reads an MPU-6050 on a breakout board ten times a
second and prints the acceleration in milli-g, the rate of turn in
milli-degrees per second and the temperature in thousandths of a degree
Celsius over the board's UART. The breakout's SDA and SCL go to the
board's I2C pins and its AD0 pin to ground, which puts the part at 0x68.
With the board flat and still, `acc` shows about 1000 on its third axis:

```
acc mg 12 -4 1003  gyr mdps 8 -15 3  temp mC 26125
```

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_imu.{ Imu }
use dev_bus
use mpu6050.{ Mpu6050 }

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    // The part answers at 0x68 on the board's first I2C bus (AD0 low).
    let imu = Mpu6050.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 },
                             dev_bus.wiring_i2c(0, 0x68))
    // Resets the part, checks WHO_AM_I and wakes it; it boots asleep.
    if not imu.begin()
        hal.uart.write("mpu6050 did not start\n")
    while true
        match imu.imu_read()
            Some(s) =>
                // s.accel.z is close to 1000 with the board flat.
                hal.uart.write(str.from_int(s.accel.z))
                hal.uart.write("\n")
            None => hal.uart.write("no answer\n")
        hal.timer.delay_ms(100)
```

The full program, with the gyroscope and the temperature, is
`examples/imu_reader.nv`. A board package that depends on this package
builds it as its `main`. `tests/device_build.sh` does so from a scratch
package for the two boards below and for the emulated board, where it runs
the image and checks that the program reports that the part did not start:

```sh
bash tests/device_build.sh
```

| Image | Flash | Of which program | Of which tables and data | RAM (`.data` and `.bss`) |
|---|---|---|---|---|
| `nrf52840-dk` | 15,832 B | 5,236 B | 848 B | 3,236 B |
| `nrf54lm20-dk` | 17,296 B | 5,676 B | 1,804 B | 3,236 B |

The numbers are from the build's memory report. Flash also holds the
runtime and the compiler's support library. The nRF52840 DK's I2C bus is
TWIM0 on SDA P0.26 and SCL P0.27, and the nRF54LM20 DK's is TWIM22 on SDA
P1.03 and SCL P1.04 (`docs/hal/README.md` in the novo-lang repository,
"The Nordic DKs' buses"), so both images carry the reading path. A reply
from an MPU-6050 over those buses is NOT VERIFIED until the driver rig.
`BoardI2c` reaches the bus that the board's package names with
`@hal(i2c)`; on a board that names none, the bus is absent hardware, every
transfer is refused, and the program prints `mpu6050 did not start`.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and
nothing else. The driver performs no input or output of its own and
allocates nothing of its own: every buffer is a fixed-size `Vec` in the
caller's frame. The bus functions of devices-nv build one short frame per
transaction inside a `system` block and release it when the call returns.
The waits go through the board's timer, which is `[hw]` too. The package
checks at every tier, `embedded` included.

## What it is

The MPU-6050 holds a 16-bit three-axis gyroscope, a 16-bit three-axis
accelerometer and a temperature sensor in a 4 x 4 mm package, with a
Digital Motion Processor and an auxiliary I2C master for external sensors.
The gyroscope measures within a full scale of ±250, ±500, ±1000 or ±2000
degrees per second (dps), the accelerometer within ±2, ±4, ±8 or ±16 g (1 g
is the acceleration of gravity, 9.81 m/s²). Each axis is a signed 16-bit
count, high byte first, and the sensitivities are the product
specification's (§§ 6.1 and 6.2):

| Accelerometer | ±2 g | ±4 g | ±8 g | ±16 g |
|---|---|---|---|---|
| Counts per g | 16,384 | 8,192 | 4,096 | 2,048 |

| Gyroscope | ±250 dps | ±500 dps | ±1000 dps | ±2000 dps |
|---|---|---|---|---|
| Counts per dps | 131 | 65.5 | 32.8 | 16.4 |

The part comes up asleep (PWR_MGMT_1 reads 0x40 after a reset) on its
internal oscillator. The register map recommends a gyroscope as the clock
reference, and `begin` selects the PLL with the X gyroscope.

The data registers are in the order accelerometer, temperature,
gyroscope, so one burst of fourteen registers from 0x3B is one sample.

The temperature is `TEMP_OUT / 340 + 36.53` degrees Celsius (register map
§ 4.19), which the driver computes as `36530 + TEMP_OUT * 1000 / 340`
milli-degrees. The product specification gives the same line as 340 LSB per
degree and an offset of -521 LSB at 35 degrees.

The sample rate is the gyroscope output rate divided by `1 + SMPLRT_DIV`.
The gyroscope output rate is 8 kHz when the digital low-pass filter is off
(DLPF_CFG 0) and 1 kHz when it is on (1 to 6). The accelerometer samples at
1 kHz, so above 1 kHz the same accelerometer sample is output more than
once. One divider serves both sensors, so a rate set for one is the rate of
the other.

| DLPF_CFG | 0 | 1 | 2 | 3 | 4 | 5 | 6 |
|---|---|---|---|---|---|---|---|
| Accelerometer bandwidth (Hz) | 260 | 184 | 94 | 44 | 21 | 10 | 5 |
| Gyroscope bandwidth (Hz) | 256 | 188 | 98 | 42 | 20 | 10 | 5 |
| Gyroscope output rate (kHz) | 8 | 1 | 1 | 1 | 1 | 1 | 1 |

The MPU-6050 is an I2C part. It answers at 0x68 with AD0 low and 0x69 with
AD0 high, at 400 kHz at most; a multiple-byte read advances the register
address. WHO_AM_I holds the upper six bits of the I2C address and so reads
0x68 at either address. The SPI interface belongs to the MPU-6000, the
sibling part, and is not on this one.

## Install

```sh
novo pkg add mpu6050-nv
```

## What the package contains

| Module | What is in it |
|---|---|
| `mpu6050` | The type `Mpu6050` (I2C), `Mpu6050Mode`, the trait `Mpu6050Port`, the bus framing, the register access, the conversions, the sample rate and its filter, the power modes, the FIFO, the self-tests with the factory trims and the bring-up |

## How to choose an entry point

A program on a board uses `Mpu6050`, built by `attach` with the board's
I2C bus, its GPIO and the wiring, as a board's device table builds it. It
implements `Accelerometer`, `Gyroscope`, `Imu`, `Thermometer` and
`Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr`; nothing is sent |
| `begin()` | brings the part up, see below |
| `accel_read()`, `gyro_read()`, `imu_read()` | `Accelerometer`, `Gyroscope`, `Imu`: milli-g, milli-degrees per second, and both from one burst of fourteen registers |
| `accel_ready()`, `gyro_ready()` | DATA_RDY_INT of INT_STATUS, one flag for both sensors, cleared by the read |
| `accel_range()`, `accel_set_range(r)`, `gyro_range()`, `gyro_set_range(r)` | the full scales, changed without touching the self-test bits |
| `accel_rate_hz()`, `accel_set_rate_hz(hz)`, `gyro_rate_hz()`, `gyro_set_rate_hz(hz)` | the sample rate in whole hertz, shared by both sensors; the slowest rate at or above `hz`; 0 puts the sensor's three axes in standby |
| `rate_mhz()`, `set_rate_mhz(mhz)`, `dlpf()`, `set_dlpf(cfg)` | the sample rate in thousandths of a hertz, and the digital low-pass filter |
| `mode()`, `set_mode(m)`, `set_wake_rate(ctrl)`, `set_clock(sel)`, `set_temp_enabled(on)` | sleep, run and cycle, the wake-up rate of cycle mode, the clock source and TEMP_DIS |
| `temp_read()` | `Thermometer`: the die temperature |
| `device_id()` | `Identifiable`: WHO_AM_I, 0x68 |
| `fifo_begin(with_gyro)`, `fifo_end()` | turn the FIFO on with the accelerometer, and the gyroscope when asked, and off |
| `accel_fifo_count()`, `accel_fifo_read(dst)`, `fifo_take_imu(accel, gyro)` | the FIFO: the sample count, the accelerometer's samples, and both sensors' |
| `route_int(mask)`, `configure_pin(active_low, open_drain)`, `status()` | the interrupt pin and INT_STATUS |
| `accel_test()`, `gyro_test()`, `accel_trim()`, `gyro_trim()` | the self-test responses and the factory trims they are compared with |
| `reg(r)`, `set_reg(r, v)` | one register, and a write that is read back |

`begin` resets the part with DEVICE_RESET and waits 100 ms, polls
PWR_MGMT_1 until it reads its reset value 0x40, checks WHO_AM_I, selects the
PLL with the X gyroscope as the clock and clears SLEEP, waits 50 ms, takes
every axis out of standby, turns the FIFO and the I2C master off, sets the
digital low-pass filter to 44 Hz and the sample rate to 100 Hz, ±2 g and
±250 dps, puts the INT pin push-pull and active high, and enables the
data-ready flag. It answers `false` for a part that does not answer 0x68
and for a register that does not hold what was written.

A program that holds its own bus, or a test with a double of the part,
implements `Mpu6050Port` over its bus and calls the module's functions,
which take the port: `mpu6050.imu_read_of(port)`, `mpu6050.start(port)`.
The framing is `i2c_read_into` and `i2c_write`, generic over `I2cBus`.

### The FIFO

The FIFO is an option. It is 1024 bytes and `begin` leaves it off.
`fifo_begin` resets it, chooses what it holds and turns it on. It holds the
accelerometer's six bytes, and the gyroscope's six too when asked, at the
sample rate, in the order of the registers, so a sample is 12 bytes for both
sensors and 6 for the accelerometer. The part has no watermark and no
packet header: the program reads the count, `accel_fifo_count()`, which is
the byte count of FIFO_COUNT divided by the bytes of a sample, or takes the
overflow flag to the pin.

When the FIFO is full the oldest byte is lost, and 1024 is not a multiple of
12 or of 6, so the bytes that remain no longer start at a sample. A count of
1024 is therefore an overflow: the reading functions empty the FIFO, which
resets it with FIFO_EN clear and sets FIFO_EN again, and answer 0 samples.
The reading takes one sample per transaction from FIFO_R_W. A FIFO set to
another layout (a temperature byte, a different set of gyroscope axes, an
external sensor) reads as empty.

## The rules a user needs

1. The part boots asleep. `begin` wakes it; a program that wakes it by
   hand writes PWR_MGMT_1 with SLEEP clear, and selects a gyroscope as the
   clock reference.
2. Wait for the part after a device reset: the Rust crate waits 100 ms and
   `begin` does the same, polling PWR_MGMT_1 for 0x40 after it. After the
   wake the PLL settles in 1 to 10 ms and the gyroscope's zero-rate output
   in 30 ms (product specification § 6); `begin` waits 50 ms.
3. WHO_AM_I is 0x68 for the MPU-6050 at both addresses. The MPU-6500,
   MPU-9250 and clones answer other values and have other register maps, so
   `begin` accepts only 0x68.
4. Reading INT_STATUS clears its flags. `accel_ready` and `gyro_ready` read
   the one DATA_RDY_INT flag, so the second of two calls in a row answers
   `false` until the next sample. A program that wants both reads `status()`
   once.
5. The sample rate is one for both sensors. `accel_set_rate_hz(hz)` sets
   the divider and takes the accelerometer out of standby; the gyroscope
   follows the same divider. A rate of 0 puts only that sensor's axes in
   standby. The rate follows the filter: a divider that gives 100 Hz with
   the filter on gives 800 Hz with it off.
6. The accelerometer-only low-power mode is `Mpu6050Cycle`: CYCLE set,
   SLEEP clear, the temperature sensor off and the gyroscope axes in
   standby, as the register map lists the steps (§ 4.31). The part wakes at
   the rate of `set_wake_rate` (1.25, 5, 20 or 40 Hz), takes one sample and
   sleeps.
7. The self-test response is the output with the self-test enabled minus
   the output with it disabled (register map § 4.1). The driver puts the
   gyroscope at ±250 dps and the accelerometer at ±8 g, as the register map
   says, passes ten samples, averages five with the self-test off and five
   with it on, and answers the difference. `gyro_trim()` and `accel_trim()`
   compute the factory trim from SELF_TEST_X to SELF_TEST_A with the
   register map's formulas, in the same units, and `gyro_self_test_ok` and
   `accel_self_test_ok` check that every axis is within ±14 % of it, the
   limit of the product specification for parts of revision D and later. A trim of 0 (a test value of 0)
   cannot be compared and fails. The board must be still.
8. The accelerometer's high-pass filter bits are not in this revision of
   the register map; the filter is not configured.
9. `device_id` answers `None` for a bus that no part drives: it reads 0xFF,
   or 0x00 when held low.

## What is not included

- The Digital Motion Processor and its firmware. NOT IMPLEMENTED. The
  firmware is InvenSense's MotionApps software.
- The auxiliary I2C master (I2C_MST_CTRL and the slave registers 0x25 to
  0x35, EXT_SENS_DATA), the pass-through bypass, and the external sensors in
  the FIFO. NOT IMPLEMENTED. The registers are reachable through `reg` and
  `set_reg`.
- The motion-detection interrupt (MOT_THR, MOT_DETECT_CTRL), the frame sync
  pin (EXT_SYNC_SET), and the INT pin's latch and clear modes. NOT
  IMPLEMENTED.
- The MPU-6000, which has the same registers and an SPI interface, and the
  MPU-6500 and MPU-9250.
- The async form of the traits. NOT IMPLEMENTED: devices-nv's
  `AccelerometerAsync`, `GyroscopeAsync` and `ThermometerAsync` are
  `async fn` traits, which `@tier(embedded)` refuses (E4002) and which take
  no `@value` struct, and the board handles are `@value` structs. The part
  completes nothing in the background that a program waits for: it samples
  on its own clock and the program reads the data-ready flag or the line.
  The interrupt line as `AsyncDeviceReady` needs an edge source on
  `BoardGpio` that embedded-hal-nv does not have.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  traits this driver implements, the wiring record and the register
  helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus` and `GpioOut` traits and the board handles.
- icm42688-nv drives TDK InvenSense's ICM-42688-P, the current part in the
  same line, on I2C and SPI.
- bmi270-nv and lsm6dso-nv drive six-axis IMUs from Bosch Sensortec and ST,
  with the same traits.

## Tests

`novo test` runs the module over a double of the part that answers on an
I2C bus: the register address advance and the FIFO port that does not
advance it, WHO_AM_I and the values of other parts, every conversion at
every range of both sensors, the temperature at the product specification's
offset, the sample rate at both output rates with the divider's limits, the
standby of each sensor, the power states, the wake-up rate and the clock
choice, the interrupt pin, the FIFO with 12-byte and 6-byte samples, a
layout that is not read, an overflow and a full list, the factory-trim
tables and their registers, the self-tests with each write refused in turn,
the bring-up with each write refused in turn, and the type over the host's
board handles. The log of bytes sent is compared for each entry point. A
second port refuses a chosen write or read of a chosen register, which
reaches the refusals a read-only register cannot. The suite covers every
line of `src/`, and `tests/coverage.sh` measures it:

```sh
NOVO=bin/novo bash tests/coverage.sh
```

**Hardware: NOT VERIFIED.** See the device program section.

The sources read for this driver, for their fixes and not ported, are the
two documents above, Zephyr's `drivers/sensor/tdk/mpu6050` and the Rust
crate `mpu6050` 0.1.6 (`juliangaal/mpu6050`). Embassy and esp-hal hold no
driver for the part. Each fix kept is named in `src/mpu6050.nv` where it is
kept:

- The crate wakes the part with PWR_MGMT_1 set to the PLL with the X
  gyroscope and waits 100 ms; Zephyr clears SLEEP only. `begin` takes the
  crate's clock and waits.
- The crate notes that a device reset sets SLEEP again (the reset value of
  PWR_MGMT_1 is 0x40) and waits 100 ms; `begin` polls for 0x40 after the
  wait.
- The crate checks WHO_AM_I against 0x68 only; Zephyr also accepts the
  values of the MPU-6500 and the MPU-9250, parts of another register map,
  which `begin` refuses.
- The crate sets and clears single bits; so do the setters, which keep
  INT_PIN_CFG's bypass bit and the self-test bits.
- The register map says the oldest data is lost on an overflow and the count
  is in bytes, which is why a full FIFO is emptied.

## Licence

Apache-2.0.
