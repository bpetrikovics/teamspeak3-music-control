# MojoBot - Teamspeak bot + Mopidy remote control with simple web UI #

## What it does:

* Runs on a Linux box, either physical or a virtual machine (tested with Debian 8 and VirtualBox)
* Uses the stock TS3 client you download from the Teamspeak website
* Can play either local music or anything via Google Play (All access)
* Can be remote controlled via a very simple web UI

## Components used:

* Pulseaudio - to provide the "link" between mopidy's output and teamspeak's input, as a virtual soundcard
* Mopidy - music player daemon, extensible with plugins
* Xvfb - headless X display server, needed by teamspeak
* x11vnc - to give remote connectivity to Xvfb, mostly for setup

## Optional, for Google Play Music:

* gmusicapi (via pip)
* Moopidy-Gmusic (via pip)

## How to get it working:

Clone MojoBot into subdirectory of your liking.

Download Teamspeak from http://www.teamspeak.com/downloads (Linux client, 32 or 64-bit according to your host's architecture)

Execute the downloaded file and let it install itself. It will create a directory for itself, for example "TeamSpeak3-Client-linux_amd64"

Install components with apt-get:

    sudo apt-get install pulseaudio xvfb x11vnc python-pip mpc

Install mopidy, see: https://docs.mopidy.com/en/latest/installation/debian/

Optionally, for Google Music:
    sudo pip install Mopidy-Gmusic
    sudo pip install Mopidy-Gmusic-Webclient

Start mopidy once so it creates a default configuration; then kill it and edit the config file under .config/mopidy/

* Enable the http module and set the listen address to a value that lets you to reach it (0.0.0.0 would instruct it to listen on all interfaces). Make sure you protect this port with appropriate firewall settings and only allow access for yourself.
* Set up your media location ([local]) and/or Google Music credentials and settings, whichever you want to use.
* For mopidy-gmusic, you can find configuration details here: https://github.com/mopidy/mopidy-gmusic
* If you are using locally stored music, run (only once) "mopidy local scan"

At this time, you will need to create your pulseaudio configuration, which you will add to your startup script. You will need to take note of some values, as these can be different in each installation, based on the given host's audio devices.

    pactl stat
    pactl load-module module-null-sink sink_name=Virtual1
    pactl load-module module-loopback sink=Virtual1

The above should create a virtual audion sink (output) and source (input). Now list the devices, and take note of the number of the newly created Sink and Source devices:

    pactl list sinks
    pactl list sources

Use the numer you gathered above to set those as the default Sink and Source. Substitute X and Y with the corresponding number.

    pactl set-default-sink X
    pactl set-default-source Y

Now create a simple startup script that will do these (and other initialization stuff) for you as soon as the server comes up. I use the below script.
Please note that you will need to replace the SINK and SOURCE variables with your own values (the ones you used in the above commands). Don't worry anout reboots, when the script runs on a freshly booted server, the newly created devices should always get the same numbers (at least, in my case, they do).

    #!/bin/bash

    export SINK=1
    export SOURCE=2

    cd

    # This will make sure pulseaudio is running
    /usr/bin/pactl stat
    
    # Sink/Source config
    pactl load-module module-null-sink sink_name=Virtual1
    pactl load-module module-loopback sink=Virtual1
    pactl set-default-sink $SINK
    pactl set-default-source $SOURCE

    # Turn down volume a bit
    /usr/bin/pacmd set-sink-volume $SINK 35000 [later on, replace 0 with the number of your actual output sink]

    /usr/bin/mopidy &
    sleep 5

    /usr/bin/mpc random on
    /usr/bin/mpc volume 100

    /usr/bin/Xvfb :20 -screen 0 1280x1024x24 -cc 4 -nolisten tcp &
    /usr/bin/x11vnc -display :20 -forever &

    cd MojoBot && /usr/local/bin/hypnotoad MojoBot

The above also assumes that you cloned the MojoBot repository into a directory called MojoBot in the bot user's home directly. You should adjust this if it's not the case.

You can (actually, should) add this to the crontab:

    crontab -e

This should work as crontab setting:

    # m h  dom mon dow   command
    @reboot /path/to/startit > /tmp/start.out 2>&1

Now, you'll need to create the initial Teamspeak config. Rin the start script manually once so you have the dummy X server and the VNC server running, and connect to it with any VNC client you line (I use one from the Chrome app store).

At the beginning you will see a black screen and some warnings about lack of authentication. Then, from an SSH session, start TS3 manually:

    DISPLAY=:20 ./tsclient_linux_amd64.sh

At this point, the TS client should appear in your VNC screen. Click through any license dialogs and then:

* Create a bookmark for the server you want to connect to. You will refer to it by the server address later.
* Configure TS to use the virtual audio:
  * Settings->Playback and Capture: Playback/Capture mode: Pulseaudio, default device
  * Set capture mode to Voice Activation Detection, and set the threshold somewhere between -50 and -40
  * Unseleft Echo reduction
  * Under Advanced, unselect Remove background noise
   