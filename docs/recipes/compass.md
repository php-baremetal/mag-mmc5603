---
eyebrow: 'mag-mmc5603 · Recipe'
lede:    'A compass on the serial console: heading in degrees, the nearest cardinal point and the field, a few times a second.'
see_also:
  - { href: '../usage/php-api.md', meta: 'PHP API' }
  - { href: 'package:php-baremetal/ext-i2c#docs/recipes/scan-the-bus.md', meta: 'ext-i2c', label: 'Scan the bus' }
prev: { label: 'Mmc5603', href: '../usage/php-api.md' }
next: { label: 'No next page', href: '#' }
---

# A serial compass

An MMC5603 module on the extra I2C connector of a Waveshare ESP32-S3-Touch-LCD-1.85B (SDA 7 / SCL 8). On
another board, use any two free pins and keep the module away from speakers, motors and steel screws.

## The project

<!-- @code-block language="json" label="composer.json" -->
```json
{
  "name": "acme/compass",
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

## The script

<!-- @code-block language="php" label="project-src/index.php" -->
```php
<?php
use Baremetal\I2c\Bus;
use Baremetal\I2c\Driver\Mmc5603;

$mag = new Mmc5603(new Bus(sda: 7, scl: 8));

function cardinal(float $deg): string
{
    return ['N', 'NE', 'E', 'SE', 'S', 'SW', 'W', 'NW'][(int) round($deg / 45) % 8];
}

function loop(): void
{
    global $mag;

    [$x, $y, $z] = $mag->mag();
    $h = $mag->heading();
    printf("heading %5.1f deg  %-2s   field [% 6.1f % 6.1f % 6.1f] uT\n",
        $h, cardinal($h), $x, $y, $z);
    usleep(300000);
}
```
<!-- @endcode-block -->

## How it works

- Hold the board flat and turn it: `heading()` follows. It is measured from the chip's x axis, so "N" is
  where the x axis points when the reading is 0; mark that on the board once. If east and west come out
  swapped for the way the chip is mounted, use `360 - $h`.
- `mag()` and `heading()` each read the chip, so this loop does two transactions per line. To read once, compute
  the heading from `mag()` yourself: `fmod(rad2deg(atan2($y, $x)) + 360, 360)`.
- The Earth's field is roughly 25 to 65 µT. A much larger magnitude means a magnet or steel is close by.
