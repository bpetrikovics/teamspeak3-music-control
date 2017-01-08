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



