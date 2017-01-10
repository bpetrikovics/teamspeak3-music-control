# TS3 Music Control - Teamspeak bot / Mopidy remote control #

Web frontend to control TS3 musicbot on Linux

## What is this?

This is not a musicbot per se, but rather a Web frontend with a quick howto that explains the process of setting up
a stock Teamspeak client to act as a remote-controlled music player.

The interface itself is written in Perl using the Mojolicious framework running on Linux. You could probably use almost any
distribution, I made it for and tested on Debian 8. Should work perfectly in virtual machines as well, so you can also run this
on your Windows desktop in Virtualbox to have your bot in a nice and contained environment.

The idea and the original setup came from these howtos:

* http://forum.teamspeak.com/threads/101054-GUIDE-WALKTHROUGH-Music-bot-without-GUI-or-soundcard-ideal-for-VPS-server
* http://haydenivey.com/blog/?p=223

## What it does:

* Runs on a Linux box
* Manages a stock TS3 client you download from the Teamspeak website; stops and starts it as and when needed
* Plays music via Mopidy (a variant of MPD, the Music Player Daemon), local from local mp3 files music or anything via Google Play (All access); you just need to have a playlist defined
* Re-routes the output of Mopidy to the input/source of Teamspeak
* Gives you a simple interface to control the above

## ..And what it does NOT do (yet.. TBD..)

* Access control - the assumption is that you would not directly put this on the internet without protection, and use some security measures to protect it (e.g. firewall; access via VPN; apply basic HTTP user/password protection with a htpasswd file; run in a VM which is not publicly accessible)

## Components used:

* Pulseaudio - to provide the "virtual audio circuit"
* Mopidy - music player daemon, extensible with plugins
* Xvfb - headless X display server, needed by teamspeak
* x11vnc - to give remote connectivity to Xvfb, mostly for setup

## Optional, for Google Play Music:

* gmusicapi (installed via pip)
* Moopidy-Gmusic (installed via pip)

## The Howto:

The below will assume a basic understanding of Linux, commandline, package installation including pip and CPAN,
and configuring a webserver. You also need to have root access to the Linux box on which you are installing.

Create a user for the bot

    sudo adduser musicbot

Use "su" or "sudo" to switch to the newly created ID, then clone the "Bot" into subdirectory of your liking.
This howto assumes you used the MojoBot directory in the musicbot user's home.

    git clone http://....... 

Download Teamspeak from http://www.teamspeak.com/downloads (Linux client, 32 or 64-bit according to your host's architecture)

Execute the downloaded file and let it install itself. It will create a directory for itself, for example
"TeamSpeak3-Client-linux_amd64"

Install components with apt-get:

    sudo apt-get install pulseaudio xvfb x11vnc python-pip mpc libmojolicious-perl libaudio-mpd-perl libdir-self-perl

Mojolicious::Plugins::AccessLog doesn't appear to be available in as debian package (correct me if I'm wrong),
so get it from CPAN:

    sudo perl -MCPAN -e shell
    cpan[1]> install Mojolicious::Plugins::AccessLog

Install mopidy using instructions on its website: https://docs.mopidy.com/en/latest/installation/

Install the Web frontend for Mopidy:

    sudo pip install Mopidy-MusicBox-Webclient

Optionally, for Google Music, you will need this one as well:

    sudo pip install Mopidy-Gmusic

Start mopidy once so it creates a default configuration; then kill it and edit the config file under .config/mopidy/

* Make sure the [http] module is enabled and set the listen address to a value that lets you reach it (0.0.0.0 would instruct it to listen on all interfaces). Make sure you protect this port with appropriate firewall settings and only allow access for yourself.

```
    [http]
    #enabled = true
    hostname = 0.0.0.0
    #port = 6680
    #static_dir =
    #zeroconf = Mopidy HTTP server on $hostname
```

* Make sure the mpd component is enabled and listening

```
    [mpd]
    #enabled = true
    #hostname = 127.0.0.1
    #port = 6600
```

* Set up your media location ([local]) and/or Google Music credentials and settings (in the [gmusic] section), whichever you want to use.

```
    [local]
    enabled = true
    library = json
    media_dir = /path/to/your/local/music
    playlists_dir = /paht/to/your/playlists
```

* For mopidy-gmusic, you can find configuration details here: https://github.com/mopidy/mopidy-gmusic

```
    [gmusic]
    enabled = true
    username = yourloginhere
    password = appspecificpassword
    # 128, 160, 320
    bitrate = 128
    deviceid = yourhexdeviceid
    all_access = true
    refresh_library = 1440
    refresh_playlists = 60
    radio_stations_in_browse = true
    radio_stations_as_playlists = true
    radio_stations_count = 15
    radio_tracks_count = 50
```

* If you are using locally stored music, run (only once) "mopidy local scan" which will look for your files and build the media library.

At this time, you will need to create your pulseaudio configuration, which you will add to your
startup script. You will need to take note of some values, as these can be different in each installation,
based on the given host's audio devices.

    pactl stat
    pactl load-module module-null-sink sink_name=Virtual1
    pactl load-module module-loopback sink=Virtual1

The above should create a virtual audion sink (output) and source (input). Now list the devices, and take note
of the device IDs of the newly created Sink and Source devices:

    pactl list sinks
    pactl list sources

You will see something like this:

```
Sink #0
        State: IDLE
        Name: auto_null
        Description: Dummy Output
        Driver: module-null-sink.c
        Sample Specification: s16le 2ch 44100Hz
```

```
Source #0
        State: IDLE
        Name: auto_null.monitor
        Description: Monitor of Dummy Output
        Driver: module-null-sink.c
        Sample Specification: s16le 2ch 44100Hz
```

In this case, both sink and source devices got the "0" ID.

Use the number you gathered above to set those as the default Sink and Source. Substitute X and Y with the
corresponding number.

    pactl set-default-sink X
    pactl set-default-source Y

Now create a simple startup script that will do these (and other initialization stuff) for you as soon as the
server comes up. I use the below script, I call it "startit". Please note that you will need to replace the
SINK and SOURCE variables with your own values (the ones you used in the above commands). Don't worry about
reboots, when the script runs on a freshly booted server, the newly created devices should always get the
same numbers (at least, in my case, they do).

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
    /usr/bin/pacmd set-sink-volume $SINK 35000

    /usr/bin/mopidy &
    sleep 5

    /usr/bin/mpc random on
    /usr/bin/mpc volume 100

    /usr/bin/Xvfb :20 -screen 0 1280x1024x24 -cc 4 -nolisten tcp &
    /usr/bin/x11vnc -display :20 -forever &

    cd MojoBot && /usr/bin/hypnotoad MojoBot

The above also assumes that you cloned the MojoBot repository into a directory called MojoBot in the bot
user's home directly. You should adjust this if it's not the case. And don't forget to make it executable:

    chmod +x startit

You can (actually, should) add this to the crontab with "crontab -e"

This should work as crontab setting:

    # m h  dom mon dow   command
    @reboot /path/to/startit > /tmp/start.out 2>&1

Now, you'll need to create the initial Teamspeak config. Run the start script manually once so you have the
dummy X server and the VNC server running, and connect to it with any VNC client you line (I use one from
the Chrome app store).

At the beginning you will see a black screen and some warnings about lack of authentication. Then, from an SSH
session, start TS3 manually:

    DISPLAY=:20 ./tsclient_linux_amd64.sh

At this point, the TS client should appear in your VNC screen. Click through any license dialogs and then:

* Create a bookmark for the server you want to connect to. You will refer to it by the server address later.
* Settings->Playback and Capture: set both Playback and Capture mode to Pulseaudio, leave device set to "Default"
* Set capture mode to Voice Activation Detection, and set the threshold somewhere between -50 and -40
* Unselect Echo reduction
* Unselect Automatic voice gain control
* Under Advanced, unselect Remove background noise (messes up music)

And last, look at any .dist file under the MojoBot directory, rename them (remove the .dist extension) and
customize the values/settings in them.

* MojoBot.conf.dist: this is the one you need to really customize, and set paths, what address to listen on, what port the web UI will be accessible from the outsde (in case of using a reverse proxy), what TS server to connect to and which playlist should be played when the bot starts 
* templates/player.html.ep.dist: most likely you don't need any change, unless you want to alter the HTML, the help text etc.

I also strongly suggest to put the web UI behind a reverse web proxy. I'm using nginx with the below settings,
listening on port 7000 on the server's public interface:

    upstream mojo {
        server 127.0.0.1:8080;
    }

    server {
        listen 7000;

        server_name _;

        root /var/www/html;

        access_log /var/log/nginx/access-mojo.log;
        error_log /var/log/nginx/error-mojo.log;

    location / {
        proxy_pass http://mojo;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/mojo.passwd;
        }
    }

You can notice that this is also the way I restrict the access to the bot, by a htpasswd file via nginx (as the bot does not offer any authentication out of the box yet).

You're done. Now you can:

* Launch mopidy and check its webinterface at http://yourhost:6680, check if it sees your media files and/or your Google Music library
* Point your browser to the http://yourhost:port/player (port is 8080 is you're not using any proxy frontend; otherwise use the external port you're forwarding)
* The UI is dead simple. Click "Start" to start TS3 and "Kill" to kill it. :)

That's all. Enjoy!
