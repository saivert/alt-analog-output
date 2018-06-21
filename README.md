Alt-analog-output
-----------------

This consists of a set of files that enables the alternative analog output on select Realtek onboard sound devices.

Windows 10's inbox drivers uses this for front panel headphone jacks but the ALSA devs have not added proper support for autodetecting this so it needs a bit of handywork.


Installation instructions
-------------------------
1. Clone this repository to a temporary directory

2. You need to enable the hint for the alsa driver.
    Tip: to get the correct vendor id and subsystem id do:

        $ grep -e 'Vendor Id' -e 'Subsystem Id' /proc/asound/PCH/codec#0

    On my system it returns something like:

        Vendor Id: 0x10ec1220
        Subsystem Id: 0x18491220

    Now edit the file `lib/firmware/hda-init.fw` in the repository and make sure your own vendor id and subsystem id are present in the codec section:

        [codec]
        0x10ec1220 0x18491220 0

        [model]
        auto

        [hint]
        indep_hp = true

    The file `etc/modprobe.d/hda.conf` in the repository will make sure this file is loaded.

3. Create an udev rule file to select pulseaudio profile-set for this card. The subsystem ID you got earlier can be broken up into two WORDS where the first is the `subsystem_vendor` and the second is the `subsystem_device`.

        UBSYSTEM!="sound", GOTO="pulseaudio_end"
        ACTION!="change", GOTO="pulseaudio_end"
        KERNEL!="card*", GOTO="pulseaudio_end"

        ATTRS{subsystem_vendor}=="0x1849", ATTRS{subsystem_device}=="0x1220", ENV{PULSE_PROFILE_SET}="analog-alt.conf"

        LABEL="pulseaudio_end"

4. Copy all the files to your root (/) and reboot.

5. Turn on the independend headphone jack option. (Yes I assume pulseaudio is installed)

        $ pasuspender -- amixer -c0 sset 'Independent HP',0 'Enabled'

    If you don't have pasuspender for some reason edit `~/.config/pulse/client.conf` and make sure this line is present:

        autospawn = no

    Then run

        $ pulseaudio -k && amixer -c0 sset 'Independent HP',0 'Enabled' && pulseaudio --start

    Make sure to revert ~/.config/pulse/client.conf after this unless you want to keep autospawn off.

5. Restart pulseaudio (or just log out and log back in). To do this from the command line:

        $ pulseaudio -k

    It should automatically start back up since autospawn is on (right??).

6. Now you should be able to select the Both profile in pulseaudio volume control and be able to play audio independently to front panel jacks. Or just use it as a secondary port selectable from your PC so you don't have to unplug the jack to use your speakers.

Troubleshooting
---------------

Make sure the device is listed in the output from `aplay -L` like so:

    alt:CARD=PCH,DEV=0
    HDA Intel PCH, ALC1220 Alt Analog
    Front headphones

Technical information
---------------------

This works by adding a new device to ALSA named `alt` which is referenced by a new pulseaudio profile set. udev rules make sure to select the correct pulseaudio profile set.


Todo
----
Current profile-set use the standard mixer paths which means that volume control isn't truly independent. Pulseaudio always adjust Master volume for the biggest overall volume adjustment and then adjusts the other mixer controls like Headphone/Front and PCM. This means that if Speaker volume is set too low you wont be able to adjust Headphone volume to max. This is only of consequence if you enable both at once using the Both profile.

I have custom mixer paths in the works that only affect the Front and Headphone mixer controls. This leaves Master untouched so you have to make sure this is set to max volume. I don't know how to make Pulseaudio automatically reset the Master mixer control to 100%.
