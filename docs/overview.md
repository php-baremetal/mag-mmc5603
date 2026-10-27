---
eyebrow: 'Package · Driver'
lede:    'The MMC5603NJ magnetometer as a PHP class: the field in microtesla, a flat compass heading, chip temperature, and 9-byte background samples. Registers into ext-i2c.'
see_also:
  - { href: './usage/php-api.md', meta: 'PHP API' }
  - { href: './recipes/compass.md', meta: 'Recipe' }
  - { href: 'package:php-baremetal/ext-i2c', meta: 'Extension', label: 'ext-i2c' }
  - { href: 'package:php-baremetal/ext-i2c#docs/drivers/registry.md', meta: 'Registry', label: 'The i2c_drivers registry' }
prev: { label: 'No previous page', href: '#' }
next: { label: 'Mmc5603', href: './usage/php-api.md' }
---

# MMC5603 magnetometer

mag-mmc5603 adds the class `Baremetal\I2c\Driver\Mmc5603` to the PHP on your board. Constructing it checks
that an MMC5603NJ answers, resets it and puts it in continuous measurement mode at 100 Hz; then `mag()` and
`heading()` read the latest result, and `poll()` hands the reading to the executor task.

| Property | Value |
| --- | --- |
| Class | `Baremetal\I2c\Driver\Mmc5603`, extends `Baremetal\I2c\Device` |
| Capability | `Baremetal\Sensor\Magnetometer` |
| Default address, clock | `0x30`, 400 kHz |
| Resolution | 20 bits per axis, 0.00625 µT per count |
| Units | `mag()` µT, `heading()` degrees 0 to 360, `temp()` °C |
| Background sample | 9 bytes |

## Requirements

| Requires | Why |
| --- | --- |
| `php-baremetal/ext-i2c` | The `Bus` and `Device` classes and the `i2c_drivers` registry this driver registers into. |

ext-i2c in turn requires the virtual `php-baremetal/i2c-hal`, `php-baremetal/ext-events` and
`php-baremetal/bm-common`. Conductor adds [esp32-i2c-hal](package:php-baremetal/esp32-i2c-hal) for an `esp32`
SDK; you never require it. The driver is C code with no platform calls of its own.

## Install

<!-- @steps -->
- **Install conductor once per machine**

  `composer global require php-baremetal/conductor`

- **Create the project**

  `phpbm init my-compass`

- **Require the driver**

  `composer require php-baremetal/mag-mmc5603` (ext-i2c comes with it).

- **Build and flash**

  `phpbm flash`, then `phpbm monitor` for the console.
<!-- @endsteps -->

The resulting `composer.json`, for an MMC5603 module on the extra I2C connector of a Waveshare
ESP32-S3-Touch-LCD-1.85B (SDA 7 / SCL 8):

<!-- @code-block language="json" label="composer.json" -->
```json
{
  "name": "acme/my-compass",
  "type": "project",
  "repositories": [
    { "type": "composer", "url": "https://repository.php-baremetal.com", "canonical": true }
  ],
  "require": {
    "php-baremetal/conductor": "^1.0",
    "php-baremetal/mag-mmc5603": "^1.0"
  },
  "config": {
    "allow-plugins": { "php-baremetal/conductor": true },
    "platform": { "php": "8.5.11" }
  },
  "extra": {
    "baremetal": {
      "sdk": "esp32@^2.0",
      "board": "waveshare-esp32-s3-touch-lcd-1.85b",
      "runtime": "init-loop",
      "storage": "embedded"
    }
  }
}
```
<!-- @endcode-block -->

## A first script

<!-- @code-block language="php" label="project-src/index.php" -->
```php
<?php
use Baremetal\I2c\Bus;
use Baremetal\I2c\Driver\Mmc5603;

$mag = new Mmc5603(new Bus(sda: 7, scl: 8));

function loop(): void
{
    global $mag;

    [$x, $y, $z] = $mag->mag();
    printf("[% 7.1f % 7.1f % 7.1f] uT  |B| %5.1f  heading %3.0f\n",
        $x, $y, $z, sqrt($x * $x + $y * $y + $z * $z), $mag->heading());
    usleep(500000);
}
```
<!-- @endcode-block -->

## PHP API at a glance

| Method | Returns | Meaning |
| --- | --- | --- |
| `new Mmc5603(Bus $bus, int $address = 0x30, int $hz = 400000)` | | Attach, check the chip identity, start continuous mode. |
| `mag()` | `array` | `[x, y, z]` in µT, the latest measurement. |
| `heading()` | `float` | Degrees from the x axis towards y, 0 to 360, board held flat. |
| `temp()` | `float` | Chip temperature in °C (triggers a measurement). |
| `decode(string $raw)` | `array` | `['mag' => [x, y, z], 'heading' => degrees]` from one 9-byte sample. |

The class also has every `Device` method. Details in [Mmc5603](./usage/php-api.md).

## Registration

The package's `composer.json` declares `"register": { "i2c_drivers": ["mmc5603_driver_desc"] }`. At build time
the descriptor goes into ext-i2c's driver table and at startup ext-i2c registers the class from it, so
requiring the package is all it takes. `Bus::scan()` reports `mmc5603` for a device opened through this class.
See [The i2c_drivers registry](package:php-baremetal/ext-i2c#docs/drivers/registry.md).

## Notes

- The constructor reads the product id register (`0x39`) and expects `0x10`; otherwise it throws
  `cannot init mmc5603 at 0x30: ESP_ERR_NOT_FOUND` (or the transfer error when nothing answers).
- Initialisation resets the chip, runs one SET/RESET degauss and enables continuous mode at 100 Hz; it takes
  about 40 ms.
- `heading()` uses only x and y (`atan2(y, x)`), with no tilt compensation and no hard- or soft-iron
  calibration. Nearby metal, the board's own currents and magnets shift it; calibrate in PHP from `mag()` if
  you need accuracy.
- The chip measures at 100 Hz, so reading or polling faster than that returns repeated values.
- Failed reads throw `\Exception` with `MMC5603 mag read failed` or `MMC5603 temp read failed`.
