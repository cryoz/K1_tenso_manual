# Instructions for setting up load cells on K1/K1MAX/K1C printers

## 1. Hardware part. load cells wiring

For correct operation of the load cells it is necessary to remove the bed, solder the load cells in parallel and plug them into one of the channel connector. Remember/record the channel number, the number can be seen on the last picture.

![alt text](pics/tenso_wiring1.jpg)
![alt text](pics/tenso_wiring2.jpg)
![alt text](pics/tenso_wiring3.jpg)
![alt text](pics/tenso_wiring4.jpg)
![alt text](pics/tenso_wiring5.jpg)
![alt text](pics/tenso_wiring6.jpg)
![alt text](pics/tenso_wiring7_slot.jpg)
_all photo by (c) ZeyHex_

**Record/remember which channel number the paralleled load cells were plugged into!**

**N.B.** On the K1 MAX, the board is rotated 180 degrees, not as pictured. Read the channel number either from the main large connector clockwise or refer to the inscriptions on the board:
![alt text](pics/tenso_board_slots.png)

After all operations, assemble the bed.

## 2. Firmware part.

It is necessary to build new firmware for the bed mcu, nozzle and main mcu. Either compile it yourself - or download it from [here](https://github.com/cryoz/K1_tenso_manual/tree/main/outfw)

For firmware compilation you need any Linux distro, here are the instructions using Debian as an example. 


    sudo apt install build-essential gcc-arm-none-eabi

    git clone https://github.com/cryoz/K1_Series_Klipper.git && cd K1_Series_Klipper
    ./build.sh

If everything builds without errors - the `outfw` folder with three firmwares will appear:

    bed_klipper.bin
    mcu_klipper.bin
    noz_klipper.bin

Download them to the printer in a temporary folder, e.g. `/usr/data/tenso/fw`

## 3. Setup mainline klipper

I used the [K1-klipper](https://github.com/K1-Klipper/klipper) project mostly because of its easy installation, and then modernized it with necessary modules. But then I decided to [fork](https://github.com/cryoz/klipper) this repository, adding all necessary modules from [garethky](https://github.com/garethky) and [ZeyHex](https://t.me/ZeyHex) at once

Backup the entire printer config folder (`/usr/data/printer_data/config`), you can do this via helper-script.

To install the klipper on the printer:

    cd /usr/data
    wget --no-check-certificate https://raw.githubusercontent.com/cryoz/installer_script_k1_and_max/main/installer.sh
    chmod +x installer.sh
    ./installer.sh

After the installation is complete, do NOT reboot the printer - download all three firmware files to the `/usr/data/klipper/fw/K1` folder. If the firmware files were previously downloaded to `/usr/data/tenso`:  

    cd /usr/data/tenso/fw && cp *.bin /usr/data/klipper/fw/K1/    

It is necessary to backup the log file of the firmware flasher of stock firmware in case of version or revision mismatch:

    cp /tmp/mcu_update.log /usr/data/tenso/

**N.B.** if the Helper-script was used - you need to reinstall all modules that were installed.

## 4. Printer setup

### 4.1 Backup
Backup the entire printer config folder again, you can use helper-script.

### 4.2 Config
First of all, remove all `[prtouch_v2]` `[prtouch_v1]` sections from printer.cfg.

Correct load cells settings in `/usr/data/printer_data/config/loadcell_probe.cfg` file by taking the required pins from the channel into which you soldered the resoldered load cells from this table:

| Channel      | dout      |  sclk      |
|-------------:|:---------:|:----------:|
| 1            | PA0       | PA2        |
| 2            | PA1       | PA5        |
| 3            | PA3       | PA6        |
| 4            | PA4       | PA7        |


Change to the required ones in the config file:

    [load_cell_probe]
    sensor_type: hx711
    dout_pin: leveling_mcu:PA4
    sclk_pin: leveling_mcu:PA7

Here the load cells are plugged into channel 4, the pins are configured accordingly.

Then add the load cell settings file to printer.cfg:

    [include loadcell_probe.cfg]

After that, you need to restart the printer by **powering off** - the firmware is only flashed in this case.


If the klipper gives errors about obsolete MCU - new firmware has not been flashed, look at the firmware log in `/tmp/mcu_update.log`.
If the klipper gave errors about configs - look locally: either reinstall the necessary modules via helper-script or delete stock creality sections like prtouch_v2.

If everything starts up and the klipper shows no errors, you can proceed to check the load cells and calibrate them.


### 4.3 Check

to check the functionality of the load cells, run in the console:

    $ LOAD_CELL_DIAGNOSTIC LOAD_CELL=load_cell_probe
    // Collecting load cell data for 10 seconds...
    // Samples Collected: 832
    // Measured samples per second: 83.7, configured: 80.0
    // Good samples: 832, Saturated samples: 0, Unique values: 464
    // Sample range: [0.49% to 0.50%]
    // Sample range / sensor capacity: 0.00257%

If the output is anything like this, then the load cells are working

### 4.4 Calibrate

Main docs from the author:
https://github.com/garethky/klipper/blob/adc-endstop/docs/Load_Cell.md#calibrating-a-load-cell

In short.
Remove the printer head from the work area so that it is out of the way.

Run in console:

    $ CALIBRATE_LOAD_CELL LOAD_CELL=load_cell_probe
    // Starting load cell calibration.
    // 1.) Remove all load and run TARE.
    // 2.) Apply a known load, run CALIBRATE GRAMS=nnn.
    // Complete calibration with the ACCEPT command.
    // Use the ABORT command to quit.

Then:

    $ TARE
    // Load cell tare value: 0.53% (89146)
    // Now apply a known force to the load cell and enter the force value with:
    // CALIBRATE GRAMS=nnn

Write the value `tare value`: _89146_

Next you need an item over 1kg and an accurate scale - weigh the item on the scale, memorize and place the item on the printer table. Run in console:

    $ CALIBRATE GRAMS=1828
    // Calibration value: 0.25% (42719), Counts/gram: 25.39770, Total capacity: +/- 657.07Kg
    // ERROR: Tare and Calibration readings are less than 1% different!
    // Use more force when calibrating or a higher sensor gain.
    !! ERROR: Tare and Calibration readings are less than 1% different!

Due to the peculiarities of the load cells on K1 and the implementation of the algorithm and calibration in the klipper - this error is normal. Write the value `Counts/gram`: _25.39770_

Enter the recorded values into the load cell configuration loadcell_probe.cfg:

    reference_tare_counts: 89146
    counts_per_gram: 25.39770

Write the following changes to the `[stepper_z]` section of the `printer.cfg` file.

Instead of:

    endstop_pin: tmc2209_stepper_z:virtual_endstop# PA15   

Write:    

    endstop_pin: probe:z_virtual_endstop

Save and restart klipper.

 
You can try homing, if all is successful - you can try to build a bed map.

Further fine-tuning of load cells - via graphs https://github.com/garethky/klipper/blob/adc-endstop/docs/Load_Cell.md#viewing-live-load-cell-graphs

## 5. Tweaks

### 5.1 Speed Z-Home

At the standard Z-Home speed, the nozzle is pushed into the bed with a force of 1.5kg, which is not conducive to the health of the bed, the coating or the nozzle. Add a parameter to the `[stepper_z]` section to reduce the homing speed:
    
    homing_speed: 2

### 5.2 Compensation for thermal expansion 

Nozzle expansion compensation:
https://github.com/garethky/klipper/blob/adc-endstop-k1-debug/docs/Load_Cell_Probe.md#temperature-compensation

Note - the g_code of the load cells contains a limit of nozzle temperature, above which it will refuse to work - to avoid damage to the bed coating. To set the temperature expansion compensation you should **temporarily** increase the `PROBE_TEMP` parameter in the `loadcell_probe.cfg` file:

    activate_gcode:
        {% set PROBE_TEMP = 150 %}
        {% set MAX_TEMP = PROBE_TEMP + 5 %}
        {% set ACTUAL_TEMP = printer.extruder.temperature %}
        {% set TARGET_TEMP = printer.extruder.target %}

And after adjusting, bring it back up to 140-150 degrees.

### 5.3 Температурные датчики в каждом MCU

[ZeyHex](https://t.me/ZeyHex) added to the firmware reading the temperature from each of the mcu from the built-in sensors - which made it possible to monitor the temperature on the bed, nozzle and main MCUs.
To enable temperature monitoring you need to add sensors to `printer.cfg`:


    [temperature_sensor mcu_temp]
    sensor_type: temperature_mcu
    sensor_mcu: mcu
    min_temp: 0
    max_temp: 100

    [temperature_sensor nozzle_mcu_temp]
    sensor_type: temperature_mcu
    sensor_mcu: nozzle_mcu
    min_temp: 0
    max_temp: 100

    [temperature_sensor bed_mcu_temp]
    sensor_type: temperature_mcu
    sensor_mcu: leveling_mcu
    min_temp: 0
    max_temp: 100

## 6. Credits

Authors of all modifications, algorithms and improvements:
- [ZeyHex](https://t.me/ZeyHex) 
- [garethky](https://github.com/garethky)

Used repos:
- https://github.com/Klipper3d/klipper
- https://github.com/CrealityOfficial/K1_Series_Klipper
- https://github.com/garethky/klipper/tree/adc-endstop
- https://github.com/K1-Klipper/klipper
- https://github.com/K1-Klipper/installer_script_k1_and_max

References:
- https://klipper.discourse.group/t/strain-gauge-load-cell-based-endstops/2134
- https://github.com/Klipper3d/klipper/pull/6555

___

&copy; Cryo 2024 v1.1b