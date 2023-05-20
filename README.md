# What is a time server?

Time servers (typically, as in this case, using the Network Time Protocol, or NTP) are used to synchronize clocks amongst computers within a network or across the Internet.  This is important for numerous reasons, including being able to determine with high confidence the order in which events happened which spanned multiple hosts, verifying that TLS certificates for HTTPS connections are still valid, and not making people late to catch a bus because their clock is running slow.

NTP classifies devices according to strata, with a stratum 0 device being something which directly receives data from a reliable clock such as an atomic clock or high-precision GPS signal.  Stratum 0 devices aren't connected to networks themselves, but machines which serve time onto the network from them are.  Each machine which receives time from a numbered stratum source numbers itself one higher than its source, to indicate how many intermediate devices are between the client and the time source.  As a result, the device serving NTP requests from a stratum 0 device will itself be a stratum 1 device.  Clients receiving time from the NTP server are themselves stratum 2 devices and so on.  The highest usable stratum is 15, with stratum 16 being used to indicate an unsynchronized device.

# What is the purpose of this project?

This project uses a GPS receiver hat ([this one from Uputronics](https://store.uputronics.com/index.php?route=product/product&product_id=81)), a GPS antenna ([the ANN-MB-00 from u-blox](https://www.u-blox.com/en/product/ann-mb-series)), and a Raspberry Pi 4 B to create a stratum 1, PPS-disciplined time server.  The specific GPS hat and antenna don't particularly matter so long as the module uses the GPIO pins and the antenna can connect to the hat somehow.  Some of the specific configuration might be slightly different, but the majority of it will be the same.

# But, why?

For most folks, there isn't a specific need for this.  Generally, synchronizing your clocks to a public NTP server is fine.  This is useful if you need greater precision (with the understanding that an SBC's hardware will likely have somewhat more drift between GPS polls than more expensive systems that poll from GPS sources).

# What's PPS?

The various satellite navigation constellations (GPS from the USA, Galileo from the EU, BeiDou from China, GLONASS from Russia, IRNSS/NavIC from India, and QZSS from Japan) all broadcast multiple types of data.  Some of that data includes highly-accurate time data (sourced from atomic clocks on the satellites), which is crucial for navigation to work properly.  However, because the time data is being broadcast, we can use it for other purposes, like for setting up a time server.  PPS stands for Pulse Per Second, and is pretty much exactly what it sounds like: it's a pulse broadcast from a satellite once per second.  A GPS receiver can then use those pulses and the received positioning data from the satellite to determine within micro- or nano-seconds what the current time is.  Note that PPS data does not, itself, contain data about what time it is.  It just specifies the beginning of a second.  The software using it needs a secondary source (such as the GPS data stream) to inform it what second the PPS data should be aligned to.

# Why USB GPS receivers aren't great for this

Essentially, USB adds overhead.  The data translation layer between PPS to USB and then processing that adds enough time (and varying time, too) that the accuracy is notably less than via a module designed to use the GPIO pins.  Because of this, most USB GPS receivers don't offer PPS data to the host.  If you can find one that does, it is possible to use it for this purpose, but it will have lower precision and is outside the scope of this document.

# How this all works

This works by having a time server daemon (Chrony, in this case, as it has less overhead than NTPd) pull data from GPSd and the PPS pin from the GPS hat, along with having preferences for sources to use when the GPS module doesn't yet have a lock (typically if the system is just starting up, or if the antenna is in a location where receiving GPS signals is difficult).  It also configures Chrony to use the RTC powered by the supercaps on the Uputronics GPS hat when it has no other usable time sources at the moment.  Other hats may or may not have an RTC, so that part may not be applicable depending on the hardware being used.

# Setting things up

Assuming you have the hardware set up and ready to go, the first step is to get a copy of the most recent Raspberry Pi 4 image from [the DietPi web site](https://dietpi.com) and write it to a microSD card, just as you would with any other Raspberry Pi 4 DietPi installation.

Once the Pi is booted, the following things need to be changed in `dietpi-config`:

- `Advanced Options > Serial/UART > ttyS0 console`: Off
- `Advanced Options > Serial/UART > ttyAMA0 console`: Off
- `Advanced Options > Serial/UART > ttyS0 (mini UART) device`: On
- `Advanced Options > Bluetooth`: Off
- `Advanced Options > I2C state`: On
- `Advanced Options > Time sync mode`: Custom

This will add, change, and/or uncomment the following lines in `/boot/config.txt`:

```
enable_uart=1
dtparam=i2c_arm=on
dtparam=disable-bt
```

It will also configure DietPi to not depend on syncing time from `systemd-timesyncd` for updates and other operations.  This will prevent DietPi from displaying errors when doing those things.

Additionally, the following two lines need to be manually added to `/boot/config.txt` (just putting them at the bottom works).  These are specific to the Uputronics GPS hat, so your PPS pin and whether you have a supported RTC may differ.  Also, for whatever reason, disabling the Bluetooth device tree seems to be required for GPSd to get valid data from the serial port the GPS hat is on (hence the `dtparam=disable-bt` above).

```
dtoverlay=pps-gpio,gpiopin=18
dtoverlay=i2c-rtc,rv3028
```

The above `dietpi-config` changes also adds these lines to `/etc/modules`:

```
i2c-bcm2708
i2c-dev
```

In addition to those lines, you'll need to manually add an additional line to `/etc/modules` (again, just putting it at the bottom is fine):

`pps-gpio`

These changes set the Pi up for allowing access to the GPS data over the serial (UART) port, sets up the PPS pin (which specific pin is used for PPS can vary by the specific model of GPS hat, so other hats may use a different pin number in the `/boot/config.txt` line for it), and sets up the RTC for access.

You may, optionally, want to [disable dyntick-idle mode](https://www.kernel.org/doc/Documentation/timers/NO_HZ.txt) to reduce the OS interruptions of the time services.  Doing so can increase the accuracy of the clock between PPS polls.  This is done by adding the following to the end of the string in `/boot/cmdline.txt`:

` nohz=off`

Note the space there.  `nohz=off` needs to be a separate option from the rest of what's on that line.

After that, you'll need to install some packages (this will remove the `systemd-timesyncd`, which is an NTP client only, but nevertheless conflicts with Chrony):

```
apt update
apt install gpsd chrony
```

Then you'll need to edit some config files.  The first one to modify is `/etc/default/gpsd`.  These are the important options to modify:

```
START_DAEMON="true"

# Devices gpsd should collect to at boot time.
# Serial devices need to be read/writeable, either by user gpsd or group dialout.
DEVICES="/dev/ttyAMA0"

# Other options you want to pass to gpsd
GPSD_OPTIONS="-n -s 115200"
```

These settings will enable GPSd as a daemon (`START_DAEMON=true`) and use data from the serial port the GPS hat is sending data to (`/dev/ttyAMA0`).  You could also include the PPS device (`/dev/pps0`) separated from the serial device by a space, but there's little reason to do that with GPSd when Chrony will be configured to receive PPS data directly.  The `GPSD_OPTIONS` field sets the following:

`-n`: Begin polling the devices as soon as it launches, instead of waiting for waiting for a client to connect.  Chrony will be your client, but having the polling already set up will speed things up slightly if there's a GPS lock already.

`-s 115200`: Sets the speed to open the serial port at.  This should be the default bit rate your GPS hat communicates at.  For the Uputronics GPS hat, this is 115200 bps, but many GPS devices default to 9600 bps.

There's also an additional option that can be set if you want to trust the GPS hat's RTC (if yours has one) after a power outage but before a new GPS lock happens, assuming your board (such as the Raspberry Pi 4 B) has to built-in RTC:

`-r`: Use the GPS device's time value even if there's no GPS lock.

Additionally, you can optionally set the following to disable automatically adding USB GPS devices to GPSd's data set:

```
# Automatically hot add/remove USB GPS devices
USBAUTO="false"
```

Next, you need to edit `/etc/chrony/chrony.conf` to add or set the following options:

```
# Set up local NTP server and allow devices from LAN to connect
local stratum 10
allow 192.168.1.0/24

# Use PPS as primary source and get data from NMEA sentences from GPS unit via GPSd
# PPS data is trusted and preferred so that it will be used above all other sources
refclock PPS /dev/pps0 refid PPS lock NMEA precision 1e-7 trust prefer
refclock SOCK /run/chrony.ttyAMA0.sock refid NMEA delay 0.2
```

These settings set Chrony to announce itself as a stratum 10 server if it only has its internal clock to go by (so, if there are no upstream Internet NTP servers, and if it doesn't get time from GPSd or the PPS device) and allows a given subnet to access it (your network), specified in [CIDR notation](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing#CIDR_notation).

The two `refclock` lines set up the PPS and GPS time sources.

`PPS /dev/pps0`: The device name of the PPS device for Chrony to read from.

`SOCK /run/chrony.ttyAMA0.sock`: A UNIX socket to open for GPSd to write to.  GPSd will automatically send data to the correct sockets Chrony opens as long as they are named as GPSd expects.  The socket names are based on the device names in the `DEVICES` directive in `/etc/defaults/gpsd`.

`refid PPS` and `refid NMEA`: Arbitrary identifiers (up to four ASCII characters) for Chrony to use to refer to these sources when displaying the status of lists of sources.

`lock NMEA`: Has Chrony use the `NMEA` refid as the reference for the PPS source.  Because PPS data itself doesn't include anything more than a data pulse, Chrony needs to reference it against another source to know which second to align it to.  This guarantees that it will use the GPS data as the source to align to.

`precision 1e-7`: The precision of the source.  The Uputronics GPS hat uses the u-blox M8 GPS module, which has a PPS precision of 30 ns, so setting the precision of its source to 100 ns will mean that it should be accurate, as the hardware accuracy will be better than the range we're allowing for it in software.

`delay 0.2`: Half of this value (in seconds) is used to calculate the maximum assumed error of the data source.  The NMEA spec (which is what's used for GPS data streams) specifies a resolution of 100 ms for data streams, so this source should not be assumed to be more precise than that.

`trust`: Assume this source is accurate at all times that it provides valid data, even when it deviates widely from other time sources.  This is useful for situations when other time sources aren't functioning properly.  Because PPS will always be the most accurate time source (as long as it's aligned to the correct second, which locking it to the GPS NMEA data stream should ensure), trusting it is safe.

`prefer`: Prefer sources with this directive above other sources without it.

At this point, you can enable the `gpsd` service:

`systemctl enable gpsd`

It is highly recommended to set the system time zone to UTC.  To do that, just run the following command and select `Etc/UTC` as the time zone:

`dpkg-reconfigure tzdata`

Finally, if your SBC or GPS hat has an RTC on it, you should remove the `fake-hwclock` package.

Now that everything's set up, it's time to reboot:

`shutdown -r now`

# Validating

Once the Pi comes back up after rebooting, it's time to validate that your sources are set up and working properly.  To do this, first log back into your Pi and run the following:

`chronyc sources`

This will provide output that looks like the following:

```
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
#* PPS                           0   4   377    10    -19ns[  -30ns] +/-  106ns
#? NMEA                          0   4   377    10    -69ns[  -80ns] +/-  100ms
^? ntp.myisp.net                 2   9   377   509  -2116us[-2122us] +/-   18ms
```

This lists each of the refid sources from the Chrony config along with any time servers in the config (or servers from pools in the config).  The important column is the one labeled `S`.  If a source is the one actively being used, it will have a `*` there.  If it's possible to use but not currently being used (this will happen for a little bit at startup as the GPS hat gets a lock on satellites and Chrony polls data), it will have a `?`.  If it's determined to be a "falseticker", or an unreliable clock, it will have an `x` and be ineligible for selection.

If you eventually (it may take some time, possibly several hours, depending on how long it takes to get a lock on enough satellites) see a `*` here by the PPS source, that means that Chrony has selected it and is serving time data from it.  It's good to validate it by making sure it's not too far off from other time servers listed, but the likelihood of that happening is very low.

At this point, you should be able to configure a client to use your new time server as an NTP source.  If you'd like to validate the accuracy on your client to make sure it's not wildly inaccurate, you can visit [time.is](https://time.is).  The resolution is only going to be in the millisecond range, but it's enough to verify that the synchronization worked and is providing a time that's able to be validated within the error range of an Internet request.

If you set up an RTC, you can also write the time to your it (which may be useful to do on a regular basis, as the RTC will have some amount of drift on its own) with the following:

`hwclock -w -v`

# Troubleshooting

If the PPS data source is never selected by Chrony, it's possible that it's selecting hosts from pools instead.  You can see this if it has samples it's detecting, but has the `*` by a different source.  To resolve that, comment out the `pool` directive in `/etc/chrony/chrony.conf` and add a `server` directive specifying a single NTP server:

```
# Use an external time server for when a GPS lock isn't available
server ntp.myisp.net
```

If Chrony never has any samples for the PPS source (it's always at 0), then it's likely the GPS unit isn't getting a lock.  To check this, you'll need to install the `gpsd-clients` package:

```
apt update
apt install gpsd-clients
```

Once that's installed, run `cgps` and look at the section of the screen on the right, in the column named "Use".  That indicates whether it's able to use (get a lock on) each of the satellites listed.  It's unlikely you'll be able to get a lock on every satellite, but you should be able to get a lock on several of them.  If this isn't happening (and, given some environmental circumstances, getting a lock may take some time, potentially up to a few hours), that will prevent Chrony from being able to source time data properly.  The most likely evidence of this in `cgps` is seeing satellites occasionally showing up as in use and then dropping back off, leaving no satellites in use.  Press the "Q" key to quit `cgps`.

The most likely cause of being unable to lock onto satellites is antenna placement.  Outdoors is ideal but unnecessary.  Placing it near an outside wall and away from metal to avoid signal interference or creating a partial [Faraday cage](https://en.wikipedia.org/wiki/Faraday_cage) will provide better results.  One situation which is difficult to resolve is if the antenna is in a building surrounded by lots of tall buildings, as those can prevent the GPS signals from reaching it.  For most areas, though, placing it near an outside wall is sufficient.  Additionally, some antennas perform better (are more sensitive) than others, so it is possible to compensate for unfortunate environmental circumstances by changing the antenna used.  The specific capabilities of your GPS device (whether it supports multi-band antennas, for example) can help in determining which antenna to use.

If the RTC isn't showing up, it may not be detected on the I2C bus.  You'll need to see your GPS hat's documentation for which I2C address it's on, but the Uputronics hat shows up on 

To check whether your RTC is showing up on the RTC bus, you can use `i2cdetect` to check what's on your I2C bus.  On the Raspberry Pi 4 B, this will be:

`i2cdetect -y 1`

If, for some reason, it's not installed, it can be installed with the following two packages:

```
apt update
apt install python3-smbus i2c-tools
```

This checks bus 1 (the one offered by the Pi) and skips the warning that scanning the bus may disrupt devices on it.  This will yield output that should look like the following:

```
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- 42 -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- UU -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --
```

So, at 0x52, there's a device in use by a driver, which caused `i2cdetect` to skip that address.  In this case, since the Uputronics GPS hat has the RTC at 0x52, that means that that device is in use by the driver.  If the driver is not loaded (possibly a config typo), it would show `52` instead of `UU` in that spot.

If no devices are shown at the address your RTC should be listed at, there's likely a hardware issue of some sort or your GPS hat doesn't have an RTC for some reason.

To verify the expected driver is in use, you can use `dmesg` to look for it loading:

`dmesg | grep rtc`

There should be a line there that looks something like this:

`[    2.674420] rtc-rv3028 1-0052: registered as rtc0`

Additionally, if you've saved the time to your RTC, you'll likely also see a line that looks like this:

`[    2.675849] rtc-rv3028 1-0052: setting system clock to 2023-05-19T06:58:56 UTC (1684479536)`