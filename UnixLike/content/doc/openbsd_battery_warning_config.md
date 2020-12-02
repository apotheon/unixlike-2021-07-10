+++
title = 'OpenBSD Battery Warning Config'
date = 2020-12-02
+++

Battery warnings are kind of a good thing on laptops.  There was a time when ThinkPads were made to produce an audible low battery level warning, but that time seems to have passed.

Looking into options for configuring a low battery level warning on a ThinkPad T450s running OpenBSD 6.8 led to discovery of a helpful file:

    /etc/sensorsd.conf

If it does not exist on your OpenBSD system, just create it to make this work.

A nice audio file playing the sound of a temple bell seemed appropriate for the purpose of indicating low battery charge.  It makes a nicely mellow but pretty unignorable audible alert sound.  Pick your own sound as needed.

This is the full content of the `sensorsd.conf` file:

    hw.sensors.acpibat0.watthour3:low=6.00Wh:command=/usr/bin/aucat -i /root/audio/temple_bell.wav

The ThinkPad T450s has two batteries, one internal and one more traditionally user-removable.  The sysctl utility can be used to find information about battery sensor data.  The following command is designed to whittle down the results to a manageable subset of sensor related kernel state components:

    $ sysctl hw.sensors|grep watthour
    hw.sensors.acpibat0.watthour0=20.41 Wh (last full capacity)
    hw.sensors.acpibat0.watthour1=1.02 Wh (warning capacity)
    hw.sensors.acpibat0.watthour2=0.20 Wh (low capacity)
    hw.sensors.acpibat0.watthour3=19.40 Wh (remaining capacity), OK
    hw.sensors.acpibat0.watthour4=23.20 Wh (design capacity)
    hw.sensors.acpibat1.watthour0=20.17 Wh (last full capacity)
    hw.sensors.acpibat1.watthour1=1.01 Wh (warning capacity)
    hw.sensors.acpibat1.watthour2=0.20 Wh (low capacity)
    hw.sensors.acpibat1.watthour3=19.32 Wh (remaining capacity), OK
    hw.sensors.acpibat1.watthour4=23.48 Wh (design capacity)

The `acpibat0` component indicates the internal battery, and the `acpibat1` component indicates the externally attached battery.  The important value for determining currently remaining battery power is `watthour3`, and because `acpibat0` is the last battery to drain away that is the choice for giving fair warning that there is not much time left before the laptop dies for lack of power.  This is how we get this part of the above config line in `sensrosd.conf`, indicating what sensor value the sensor monitoring daemon should watch:

    hw.sensors.acpibat0.watthour3

The `low=6.00Wh` indicates the threshold below which the alert should sound.  A low of six Watt-hours seems appropriate, as about 15% of the total design capacity of both batteries put together.

This command from the config uses the `aucat` utility to play the audio file:

    command=/usr/bin/aucat -i /root/audio/temple_bell.wav

Read `sensorsd.conf(5)` for more details on what configuration options are available.

To ensure this works, enable `sensorsd` (the sensor monitoring daemon) in `/etc/rc.conf.local` by adding this line:

    sensorsd_flags=

Yes, it's fine to have nothing after the equal sign.  What follows the equal sign is any options associated with the relevant daemon command name.  You can check what options are available in the daemon's manpage.  In the case of the sensor monitoring daemon, of course, that manpage would be `sensorsd(8)`.

It also might be worthwhile to adjust the `apmd` polling interval, because the alert will sound not only when you first cross the threshold set in `/etc/sensorsd.conf` but also periodically, about the length of the polling interval.  Just add something like `-t 32` to the `apmd_flags=` line of your `/etc/rc.conf.local`, where the number 32 is the number of seconds of the interval.  The default is once per ten minutes, if you do not set a number of seconds yourself, according to the manpage on this T450s.

You might also use the `-c` option of `sensorsd`, described in the `sensorsd(8)` manpage.  The default for `sensorsd` is 20 seconds according to the manpage here.  The actual time between incidents of the audio file playing with be a result of an interaction between these two polling periods, so if you want a settled and stable period between sounds you should probably set both periods to the same amount of time or set one of them to a multiple of the other.  Whatever values you give them, though, the sound should play at an interval of no more than the sum of the two configured intervals.

One quirk of the way `/etc/sensorsd.conf` configured events work is that, the way the audio config described here is written, the sound does not only play when battery level is below the low threshold.  It also plays once shortly after `sensorsd` starts.  This means it plays when you reboot the computer, for instance, or (re)start the sensor monitoring daemon at the command line.  To avoid this requires writing some (perhaps slightly complicated) script or two, and running that script from your `/etc/sensorsd.conf`, perhaps a daemon that manages when and what to do as battery levels get low.  Doing so requires more work, introduces more complexity, and potentially drains battery ever so slightly faster, though.  It also raises the question of why you might bother to use `sensorsd` at all, instead of just running your own daemon to handle everything.
