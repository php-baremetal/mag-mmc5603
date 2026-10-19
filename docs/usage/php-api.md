---
eyebrow: 'mag-mmc5603 · PHP API'
lede:    'Baremetal\I2c\Driver\Mmc5603: the constructor, the field, heading and temperature readings, and the 9-byte raw sample that poll() collects and decode() converts.'
see_also:
  - { href: '../recipes/compass.md', meta: 'Recipe' }
  - { href: 'package:php-baremetal/ext-i2c#docs/usage/device.md', meta: 'ext-i2c', label: 'Device' }
  - { href: 'package:php-baremetal/ext-i2c#docs/usage/polling-and-events.md', meta: 'ext-i2c', label: 'Background sampling and events' }
prev: { label: 'Overview', href: '../overview.md' }
next: { label: 'A serial compass', href: '../recipes/compass.md' }
---

# Mmc5603

`Baremetal\I2c\Driver\Mmc5603` extends `Baremetal\I2c\Device` and implements
`Baremetal\Sensor\Magnetometer`. All the `Device` methods work on it too; this page covers what the driver adds.

## Constructor

<!-- @method name="new Mmc5603(Bus $bus, int $address = 0x30, int $hz = 400000)" visibility="public" -->

<!-- @params -->
<!-- @param name="$bus" type="Bus" required -->
The bus the chip is wired to.
<!-- @endparam -->
<!-- @param name="$address" type="int" default="0x30" -->
7-bit address. The MMC5603NJ has a fixed address, so the default is the one to use.
<!-- @endparam -->
<!-- @param name="$hz" type="int" default="400000" -->
SCL clock for this device. A value of 0 or less uses 400 kHz.
<!-- @endparam -->
<!-- @endparams -->

The constructor attaches the device, checks the product id, resets the chip (20 ms), runs a SET then a RESET
pulse, sets the output rate to 100 Hz and starts continuous mode (another 20 ms). It throws `\Exception`:

| Message | Cause |
| --- | --- |
| `cannot init mmc5603 at 0x..: ESP_ERR_NOT_FOUND` | Something answered, but its product id is not `0x10`. |
| `cannot init mmc5603 at 0x..: ...` | The id read failed, for example `ESP_ERR_TIMEOUT` with no chip. |
| `address 0x.. on this bus is already used by another driver` | A raw device or another driver holds the address. |

Constructing it again with the same bus and address returns a handle to the same device without resetting it.

## Readings

<!-- @method name="Mmc5603::mag(): array" returns="array" visibility="public" -->

`[x, y, z]` field in µT: each axis is a 20-bit value, offset by the mid-scale 524288 and scaled by 0.00625 µT
per count. One 9-byte read of the latest continuous-mode result. Throws `MMC5603 mag read failed`.

<!-- @method name="Mmc5603::heading(): float" returns="float" visibility="public" -->

`atan2(y, x)` in degrees, mapped to 0 to 360. Assumes the board is flat (z up) and is not calibrated. Reads the
chip like `mag()` and throws the same message.

<!-- @method name="Mmc5603::temp(): float" returns="float" visibility="public" -->

Triggers a temperature measurement, waits for it (checking every 2 ms, at most about 100 ms), and returns
`raw * 0.8 - 75` °C. If the measurement does not finish in time the last register value is still returned.
Throws `MMC5603 temp read failed` only when the final register read fails.

## Background samples

<!-- @method name="Mmc5603::decode(string $raw): array" returns="array" visibility="public" -->

Converts one raw sample from `sample()` or `drain()` into `['mag' => [x, y, z], 'heading' => degrees]`, with
the same scaling as `mag()` and `heading()`. Throws `\ValueError` (`an MMC5603 sample is 9 bytes`) for a
shorter string.

The driver declares a sampler, so `poll()`, `sample()` and `drain()` work on this class. Each sample is the
9 output registers starting at `0x00`; each axis is 20 bits:

| Byte | Content |
| --- | --- |
| 0, 1 | x bits 19 to 4 |
| 2, 3 | y bits 19 to 4 |
| 4, 5 | z bits 19 to 4 |
| 6, 7, 8 | x, y, z bits 3 to 0, in the high nibble |

The chip updates its output at 100 Hz, so a faster `poll()` stores repeated values. The descriptor states a
200 Hz maximum; ext-i2c does not enforce it.

<!-- @code-block language="php" label="drain and decode" -->
```php
<?php
use Baremetal\I2c\Bus;
use Baremetal\I2c\Driver\Mmc5603;

$mag = new Mmc5603(new Bus(sda: 7, scl: 8));
$mag->poll(hz: 20, depth: 32);

function loop(): void
{
    global $mag;

    foreach ($mag->drain() as $raw) {
        ['heading' => $h] = $mag->decode($raw);
        printf("heading %5.1f\n", $h);
    }
    usleep(500000);
}
```
<!-- @endcode-block -->

## Capability

The class implements `Baremetal\Sensor\Magnetometer`, a marker interface ext-i2c always defines. Type a
parameter or an `instanceof` check on it to accept any magnetometer driver; the interface declares no methods.
