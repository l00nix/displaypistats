# displaypistats

displaypistats(1)                                                           displaypistats man page                                                           displaypistats(1)

NAME
       displaypistats - display Raspberry Pi Statistics on a 128x32 Mini OLED

SYNOPSIS
       A python script to display Raspberry Pi statistcis on a 128x32 Mini OLED based on the adafruit-circuitpython-ssd1306 library.

REQUIREMENTS
       This python3 script requires the Raspberry board to have I2C IO Headers and I2C enabled in raspios.

OPTIONS
       There are no options for this script currently. Changes to what is displayed on the OLED display have to made inside the script.

DESCRIPTION
       The script displays the following stats on the OLED:
       - IP Address
       - CPU Load [%]
       - Mem usage: [%] of total Memory
       - Disk usage: used/total GB [%]

SEE ALSO
       This script was created and adapted from here https://bit.ly/3rjHarP

AUTHOR
       Alexander Rau aka. loonix (alexander@rau.ca)
       Contributors: lady ada, Brent Rubell, Danny Nosonowitz
