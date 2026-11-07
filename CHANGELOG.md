# Changelog

## 1.1.0

- Added: ide/mag-mmc5603.stub, the PHP declarations of the package for IDEs (conductor copies it into vendor/php-baremetal/ide-stubs/).

## 1.0.2

- Fixed: the constructor ignored failed configuration writes, including the soft reset. The first failed
  write now makes it throw `cannot init mmc5603 at 0x..: ...`.
- Fixed: `temp()` returned the old register value when the measurement did not finish within 100 ms, and
  ignored a failed trigger. It now throws `MMC5603 temp measurement did not complete within 100 ms` or
  `MMC5603 temp trigger failed`. The chip does not measure temperature in continuous mode, so `temp()` now
  leaves continuous mode for the measurement and turns it back on.
- Changed: a negative `$hz` throws `\ValueError` (with ext-i2c 1.0.2); it used 400 kHz. `0` still does.

## 1.0.1

- Changed: the author's contact email.
- Docs: projects use `"sdk": "esp32@^2.0"`.

## 1.0.0

Initial release.
