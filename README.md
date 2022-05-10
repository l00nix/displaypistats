# displaypistats


## Overview

displaypistats is an installation package for raspios to automate the set up to display Raspberry Pi Statistics on a 128x32 Mini OLED.

![OLED Image sample](https://cdn-learn.adafruit.com/guides/cropped_images/000/001/659/medium640/oled.jpg?1515090242)

The idea was to take the manual process outlined by lady ada, Brent Rubell and, Danny Nosonowitz here https://bit.ly/3rjHarP and wrap all the individual configuration steps into one easy to install .deb package.

## Install

There are two ways to install this package one, the automated easy way and two the manual way:

###Automated install:

```bash

curl -s --compressed "https://l00nix.github.io/displaypistats/KEY.gpg" | sudo apt-key add -
sudo curl -s --compressed -o /etc/apt/sources.list.d/my_list_file.list "https://l00nix.github.io/displaypistats/my_list_file.list"
sudo apt update
sudo apt -y upgrade

```

###Manual install

1. Download the file from the Files section [here](displaypistats_1.0-0_all.deb) - for example `wget https://github.com/l00nix/displaypistats/blob/main/displaypistats_1.0-0_all.deb`
2. Install the package with `sudo dpkg -i ./displaypistats_1.0-0_all.deb` *Note:* Ignore the error message that the package failed to install.
3. Finish the installation by retrieving the required dependencies `sudo apt-get -f install` - The dependencies of this package are "python3, python3-pip, and python3-pil"
4. Reboot

The display should now display the following stats on the OLED:

- IP Address
- CPU Load [%]
- Mem usage: [%] of total Memory
- Disk usage: used/total GB [%]
	   
## Explained - What does the package actually do?

1. The package installs a python script that is responsible for displaying the stats on the OLED in `/usr/local/etc/displaypistats` (there are also a man page and copyright file that are being installed)
2. The postinst script of the package checks if the I2C setting is enabled on the Pi and if not enables it.
3. Next the postinst script checks if the `adafruit-circuitpython-ssd1306` library is installed and installs it if not.
4. Next the postinst script adds the python script in #1 to the `rc.local` startup file so the OLED starts displaying on system restart.
5. Finally the postinst scripts sets the reboot flag in the system 

## The postinst script in detail

```bash
#!/bin/bash

echo "Check if i2c is enabled and enable if not:"

I2C=`/usr/bin/raspi-config nonint get_i2c`
if [ $I2C == 0 ]; then
   echo "I2C is already enabled!"
else
   /usr/bin/raspi-config nonint do_i2c 0
   echo "I2C has been enabled!"
fi


echo "Check if adafruit-circuitpython-ssd1306 library is installed and install if not:"

/usr/bin/pip3 list | grep adafruit-circuitpython-ssd1306 &> /dev/null
if [ $? -eq 0 ]; then
   echo "adafruit-circuitpython-ssd1306 library is already installed!"
else
    echo "Installing adafruit-circuitpython-ssd1306 library!"
    /usr/bin/pip3 install adafruit-circuitpython-ssd1306
fi

echo "Adding displaypistats to startup"
NL=$(wc -l /etc/rc.local)
sed -i ${NL%% *}'i\sudo /usr/bin/python3 /usr/local/etc/displaypistats &' /etc/rc.local

echo "Telling system that reboot is required"
touch /var/run/reboot-required
echo "*** System restart required ***" > /var/run/reboot-required
```

## The python script that actually displays the stats on the OLED

```python
#!/usr/bin/python3
#https://learn.adafruit.com/adafruit-pioled-128x32-mini-oled-for-raspberry-pi/usage?gclid=CjwKCAiA1aiMBhAUEiwACw25MV4xWYX0RLMsR1z7sir4w6buxkvLe-2ag-TVw6GC75LHnzv3OEHlpRoC3lQQAvD_BwE
#https://bit.ly/3rjHarP


# SPDX-FileCopyrightText: 2017 Tony DiCola for Adafruit Industries
# SPDX-FileCopyrightText: 2017 James DeVito for Adafruit Industries
# SPDX-License-Identifier: MIT

# This example is for use on (Linux) computers that are using CPython with
# Adafruit Blinka to support CircuitPython libraries. CircuitPython does
# not support PIL/pillow (python imaging library)!

import time
import subprocess

import sys
sys.path.append(".")

from board import SCL, SDA
import busio
from PIL import Image, ImageDraw, ImageFont
import adafruit_ssd1306



# Create the I2C interface.
i2c = busio.I2C(SCL, SDA)

# Create the SSD1306 OLED class.
# The first two parameters are the pixel width and pixel height.  Change these
# to the right size for your display!
disp = adafruit_ssd1306.SSD1306_I2C(128, 32, i2c)

# Clear display.
disp.fill(0)
disp.show()

# Create blank image for drawing.
# Make sure to create image with mode '1' for 1-bit color.
width = disp.width
height = disp.height
image = Image.new("1", (width, height))

# Get drawing object to draw on image.
draw = ImageDraw.Draw(image)

# Draw a black filled box to clear the image.
draw.rectangle((0, 0, width, height), outline=0, fill=0)

# Draw some shapes.
# First define some constants to allow easy resizing of shapes.
padding = -2
top = padding
bottom = height - padding
# Move left to right keeping track of the current x position for drawing shapes.
x = 0


# Load default font.
font = ImageFont.load_default()

# Alternatively load a TTF font.  Make sure the .ttf font file is in the
# same directory as the python script!
# Some other nice fonts to try: http://www.dafont.com/bitmap.php
# font = ImageFont.truetype('/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf', 9)

while True:

    # Draw a black filled box to clear the image.
    draw.rectangle((0, 0, width, height), outline=0, fill=0)

    # Shell scripts for system monitoring from here:
    # https://unix.stackexchange.com/questions/119126/command-to-display-memory-usage-disk-usage-and-cpu-load
    cmd = "hostname -I | cut -d' ' -f1"
    IP = subprocess.check_output(cmd, shell=True).decode("utf-8")
    cmd = 'cut -f 1 -d " " /proc/loadavg'
    CPU = subprocess.check_output(cmd, shell=True).decode("utf-8")
# working    cmd = "free -m | awk 'NR==2{printf \"Mem: %s/%s\" ,$3,$2}' && free | awk 'NR==2{printf \" %.1f%%\" ,$3*100/$2}'"
    cmd = "free -m | awk 'NR==2{printf \"Mem: %.2f%% of \" ,$3*100/$2}' && free -hm | awk 'NR==2{printf \"%s\", $2}'"
    MemUsage = subprocess.check_output(cmd, shell=True).decode("utf-8")
    cmd = 'df -h | awk \'$NF=="/"{printf "Disk: %d/%d GB  %s", $3,$2,$5}\''
    Disk = subprocess.check_output(cmd, shell=True).decode("utf-8")

    # Write four lines of text.

    draw.text((x, top + 0), "IP: " + IP, font=font, fill=255)
    draw.text((x, top + 8), "CPU load: " + CPU, font=font, fill=255)
    draw.text((x, top + 16), MemUsage, font=font, fill=255)
    draw.text((x, top + 25), Disk, font=font, fill=255)

    # Display image.
    disp.image(image)
    disp.show()
    time.sleep(0.1)
```

## AUTHOR
       Alexander Rau aka. loonix (alexander@rau.ca)
       Contributors: lady ada, Brent Rubell, Danny Nosonowitz
