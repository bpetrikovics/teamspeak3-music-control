# MojoBot - Teamspeak bot / Mopidy remote control #

## What is this?

Simple remote control webinterface written with the Mojolicious framework, allowing you to run TS3 unattended on
a linux host and control it via browser. You will also find a quick and dirty howto that should help you to get
your bot up and running on a newly created Debian virtual machine.

## What it does:

* Runs on a Linux box, either physical or a virtual machine; can even be a VM on your windows desktop (tested with Debian 8 and VirtualBox)
* Uses the stock TS3 client you download from the Teamspeak website
* Can play either local music or anything via Google Play (All access); you just need to have a playlist defined 
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

Create a user for the bot

    adduser musicbot

Clone the "Bot" into subdirectory of your liking. This howto assumes you used the MojoBot directory in the musicbot user's home.

    git clone http://....... 

Download Teamspeak from http://www.teamspeak.com/downloads (Linux client, 32 or 64-bit according to your host's architecture)

Execute the downloaded file and let it install itself. It will create a directory for itself, for example "TeamSpeak3-Client-linux_amd64"

Install components with apt-get:

    sudo apt-get install pulseaudio xvfb x11vnc python-pip mpc libmojolicious-perl libaudio-mpd-perl libdir-self-perl

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

Use the number you gathered above to set those as the default Sink and Source. Substitute X and Y with the corresponding number.

    pactl set-default-sink X
    pactl set-default-source Y

Now create a simple startup script that will do these (and other initialization stuff) for you as soon as the server comes up. I use the below script, I call it "startit".
Please note that you will need to replace the SINK and SOURCE variables with your own values (the ones you used in the above commands). Don't worry about reboots, when the script runs on a freshly booted server, the newly created devices should always get the same numbers (at least, in my case, they do).

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

The above also assumes that you cloned the MojoBot repository into a directory called MojoBot in the bot user's home directly. You should adjust this if it's not the case. And don't forget to make it executable:

    chmod +x startit

You can (actually, should) add this to the crontab:

    crontab -e

This should work as crontab setting:

    # m h  dom mon dow   command
    @reboot /path/to/startit > /tmp/start.out 2>&1

Now, you'll need to create the initial Teamspeak config. Run the start script manually once so you have the dummy X server and the VNC server running, and connect to it with any VNC client you line (I use one from the Chrome app store).

At the beginning you will see a black screen and some warnings about lack of authentication. Then, from an SSH session, start TS3 manually:

    DISPLAY=:20 ./tsclient_linux_amd64.sh

At this point, the TS client should appear in your VNC screen. Click through any license dialogs and then:

* Create a bookmark for the server you want to connect to. You will refer to it by the server address later.
* Settings->Playback and Capture: set both Playback and Capture mode to Pulseaudio, leave device set to "Default"
* Set capture mode to Voice Activation Detection, and set the threshold somewhere between -50 and -40
* Unselect Echo reduction
* Unselect Automatic voice gain control
* Under Advanced, unselect Remove background noise (messes up music)

And last, look at any .dist file under the MojoBot directory, rename them (remove the .dist extension) and customize the values/settings in them.

* MojoBot.conf.dist: this is the one you need to really customize, and set paths, what address to listen on, what port the web UI will be accessible from the outsde (in case of using a reverse proxy), what TS server to connect to and which playlist should be played when the bot starts 
* templates/player.html.ep.dist: most likely you don't need any change, unless you want to alter the HTML, the help text etc.

I also strongly suggest to put the web UI behind a web proxy. I'm using nginx with the below settings, listening on port 7000 on the server's public interface:

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
* Point your browser to the host/port your bot (or the rverse proxy) listens on.
* The UI is dead simple. Click "Start" to start TS3 and "Kill" to kill it. :)

That's all. Enjoy!
