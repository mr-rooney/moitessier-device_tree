credits
=======
Thanks to the authors providing this source code. The original code can be found at https://github.com/raspberrypi/hats.git
Commit: 633f1b027228e2545f1ba53b63bb99e630ea6a7b

Be aware that some changes have been made!!

eeprom utilities
================

Compilation
-----------

host> make clean
host> make all


Usage
-----

Create the EEPROM image:
./eepmake settings.txt deviceTree.eep deviceTree.dtbo

