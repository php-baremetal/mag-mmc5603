# mag-mmc5603

A driver for the MMC5603NJ 3-axis magnetometer on I2C. It registers into ext-i2c's `i2c_drivers` registry, so
a firmware built with it has the class `Baremetal\I2c\Driver\Mmc5603`: it extends `Baremetal\I2c\Device`,
implements `Baremetal\Sensor\Magnetometer`, reads the magnetic field in microtesla and a compass heading in
degrees, and can sample the chip in the background.

The package requires [ext-i2c](package:php-baremetal/ext-i2c), so requiring the driver alone is enough;
conductor adds the I2C HAL for your SDK platform.

## Install

In a project created with `phpbm init`:

```bash
composer require php-baremetal/mag-mmc5603
```

## Example

```php
<?php
use Baremetal\I2c\Bus;
use Baremetal\I2c\Driver\Mmc5603;

$mag = new Mmc5603(new Bus(sda: 7, scl: 8)); // 0x30, 400 kHz

[$x, $y, $z] = $mag->mag();                  // microtesla
printf("field [% .1f % .1f % .1f] uT, heading %.0f deg\n", $x, $y, $z, $mag->heading());
```

## Documentation

- [Overview](docs/overview.md): requirements, install, a first project, notes.
- [PHP API](docs/usage/php-api.md): constructor, methods, units, the raw sample and `decode()`.
- [A serial compass](docs/recipes/compass.md)

## License

MIT.
