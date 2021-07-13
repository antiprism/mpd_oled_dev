# Install instructions for Volumio 2 using source

These instructions are for installing mpd_oled using source on
Volumio 2.

A binary package is also available, but may not include the very
latest code, see
[Install instructions for Volumio 2](install_volumio2_deb.md).


## Base system

Install [Volumio](https://volumio.org/). Ensure a command line prompt is
available for entering the commands below (e.g.
[use SSH](https://volumio.github.io/docs/User_Manual/SSH.html).)

## Install all dependencies

Install all the packages needed to build and run cava and mpd_oled
```
sudo apt update
sudo apt install build-essential git-core autoconf make libtool libfftw3-dev libmpdclient-dev libi2c-dev i2c-tools lm-sensors
```

## Build and install cava

mpd_oled uses Cava, a bar spectrum audio visualizer, to calculate the spectrum
   
   <https://github.com/karlstav/cava>

If you have Cava installed (try running `cava -h`), there is no need
to install Cava again, but to use the installled version you must use
`mpd_oled -k ...`.

Download, build and install Cava. These commands build a reduced
feature-set executable called `mpd_oled_cava`.
```
git clone https://github.com/karlstav/cava
cd cava
./autogen.sh
./configure --disable-input-portaudio --disable-input-sndio --disable-output-ncurses --disable-input-pulse --program-prefix=mpd_oled_
make
sudo make install-strip
```

## Build and install mpd_oled

Download and build libu8g2arm (running `make` might take two hours on a
Pi Zero)
```
git clone https://github.com/antiprism/libu8g2arm.git
cd libu8g2arm
./bootstrap
mkdir build
cd build
CPPFLAGS="-W -Wall -Wno-psabi" ../configure --prefix=/usr/local
make

```

Download, build and install mpd_oled.
```
cd ..   # if you are still in the cava source directory
git clone https://github.com/antiprism/mpd_oled_dev
cd mpd_oled_dev
./bootstrap
CPPFLAGS="-W -Wall -Wno-psabi" ./configure --prefix=/usr/local
make
sudo make install-strip
```


## System settings

Configure your system to enable I2C or SPI, depending on how your OLED
is connected.

### I2C
I use a cheap 4 pin I2C SSH1106 display with a Raspberry Pi Zero. It is
[wired like this](wiring_i2c.png).
In /etc/modules I have the line `i2c-dev` (included by default).
In /boot/config.txt I have the line `dtparam=i2c_arm=on` (included by default).

The I2C bus speed on your system may be too slow for a reasonable screen
refresh. Set a higher bus speed by adding the line
`dtparam=i2c_arm_baudrate=400000` to
/boot/userconfig.txt (or use /boot/config.txt for Volumio versions before
2.673), or try a higher value for a higher screen refresh (I use 800000 with a
25 FPS screen refresh)
```
sudo nano /boot/userconfig.txt
```

Restart the Pi after making any system configuration changes.

### SPI
I use a cheap 7 pin SPI SSH1106 display with a Raspberry Pi Zero. It is
[wired like this](wiring_spi.png).
In /boot/userconfig.txt (or use /boot/config.txt for Volumio versions before
2.673) I have the line `dtparam=spi=on`.
```
sudo nano /boot/userconfig.txt
```

Restart the Pi after making any system configuration changes.

## Configure a copy of the playing audio

*The next instruction configure MPD to make a copy of its output to a*
*named pipe, where Cava can read it and calculate the spectrum.*
*This works reliably, but has two disadvantages: the configuration*
*involves changing a Volumio system file, which must be undone*
*if Volumio is to be updated (see below); the spectrum*
*only works when the audio is played through MPD, like music files,*
*web radio and DLNA streaming. Creating a copy of the audio for all*
*audio sources is harder, and may be unreliable -- see the thread on*
*[using mpd_oled with Spotify and Airplay](https://github.com/antiprism/mpd_oled/issues/4)*

Configure MPD to copy its audio output to a named pipe
```
sudo mpd_oled_volumio_mpd_conf_install
```

**Note:** after running this command the next Volumio update will fail
with a *system integrity check* error. The change can be undone by running
`sudo mpd_oled_volumio_mpd_conf_uninstall`, then after the Volumio update
run `sudo mpd_oled_volumio_mpd_conf_install` to re-enable the audio copy.

## Set the time zone

If the mpd_oled clock does not display the local time then you may need
to set the system time zone. Set this in the UI, or run the following
command for a console based application where you can specify your location
```
sudo dpkg-reconfigure tzdata
```

## Configure mpd_oled and set to run at boot

*Note: The program can be run without the audio copy enabled, in*
*which case the spectrum analyser area will be blank*

Install a service file. This will overwrite an existing mpd_oled
service file
```
sudo mpd_oled_service_install
```

The mpd_oled program can now be run with `sudo mpd_oled_service_edit` (plus
options), and this also sets up mpd_oled with the same options as a service
to be run at boot. Rerunning `sudo mpd_oled_service_edit` with different
options will stop the current running mpd_oled and start it again with
the new options. (Test commands can also be run with `mpd_oled` (plus
options), and stopped with Ctrl-C, but ensure that no other copy of
mpd_oled is running).

The OLED configuration MUST be specified with -o, and is a list of values
and settings separated by commas. The first three parts are required, and
specify (in order) the OLED controller, model and communicatons protocol. See
[OLED configuration with option -o](https://github.com/antiprism/mpd_oled_dev#oled-configuration-with-option--o)
(or run `mpd_oled -o help`) for full details
Examples:
* Adafruit
  - `SSD1306,128X64,SPI`
  - `SSD1306,128X64,SPI,dc=24,reset=25`
  - `SSD1306,128X64,I2C`
  - `SSD1309,128X64,SPI`
  - `SSD1309,128X64,I2C`
* SH1196
  - `SH1106,128X64,SPI`
  - `SH1106,128X64,SPI,dc=24,reset=25`
  - `SH1106,128X64,I2C`

An example command, for a generic I2C SH1106 display with
a display of 10 bars and a gap of 1 pixel between bars and a framerate
of 20Hz is
```
sudo mpd_oled_service_edit -o SH1106,128X64,I2C -b 21 -g 1 -f 20 -c alsa,plughw:Loopback,1
```

Add extra controller settings to the option -o argument after the
contoller, model, and protocol parts, in the form `,setting_name=value`.

**For I2C OLEDs** you may need to specify the I2C
address, find this by running, e.g. `sudo i2cdetect -y 1` and then specify
the address with the `i2c_address` setting, e.g.
`sudo mpd_oled_service_edit -o SH1106,128X64,I2C,i2c_address=3d ...`.
If you have a reset pin connected, specify the GPIO number with the
`reset` setting, e.g.
`sudo mpd_oled_service_edit -o SH1106,128X64,I2C,reset=24 ...`.
Specify the I2C bus number, if not 1, with the `bus_number` setting, e.g.
`sudo mpd_oled_service_edit -o SH1106,128X64,I2C,bus_number=0 ...`.

**For, SPI OLEDs** you *must* specify your DC GPIO number with `dc`, and
you may need to specify your reset pin GPIO number with `reset`, e.g.
`sudo mpd_oled_service_edit -o SH1106,128X64,SPI,dc=24,reset=25 ...`.
Specify the SPI bus number, if not 0, with `bus_number` setting, and
the CS number, if not 0, with the `cs_number` setting, e.g.
`sudo mpd_oled_service_edit -o SH1106,128X64,SPI,bus_number=1,cs_number=1 ...`.

If your display is upside down, you can rotate it 180 degrees with setting
`rotate=2`, e.g.
`sudo mpd_oled_service_edit -o SH1106,128X64,I2C,rotate=2 ...`.

Once the display is working, play some music and check the spectrum display
is working and is synchronised with the music. If there are no bars then the
audio copy may not have been configured correctly. If the bars seem jerky
or not synchronized with the music then reduce the values of -b and/or -f.

If you run `sudo mpd_oled_service_edit` without options the service
file will open in an editor, allowing the full service file to be
changed, and not just the mpd_oled options.

If the mpd_oled options are valid the display will be started after
the editor is closed, and will also be configured to start a boot

Check the program is working correctly by looking at the display while
the player is stopped, paused and playing music.


### Extra commands to control the service

Commands from the following list can be run to control the service
(they do not need to be run from the mpd_oled directory)
```
sudo systemctl enable mpd_oled    # start mpd_oled at boot
sudo systemctl disable mpd_oled   # don't start mpd_oled at boot
sudo systemctl start mpd_oled     # start mpd_oled now
sudo systemctl stop mpd_oled      # stop mpd_oled now
sudo systemctl status mpd_oled    # report the status of the service
```

## Uninstall

Uninstall the mpd_oled service (just the service, not the mpd_oled programs
or documentation), and the audio copy with
```
sudo mpd_oled_service_uninstall
```
