# MojoBot - Teamspeak bot + Mopidy remote control with simple web UI #

What it does:

* Runs on a Linux box, either physical or a virtual machine (tested with Debian 8 and VirtualBox)
* Uses the stock TS3 client you download from the Teamspeak website
* Can play either local music or anything via Google Play (All access)
* Can be remote controlled via a very simple web UI

Components used:

* Pulseaudio - to provide the "link" between mopidy's output and teamspeak's input, as a virtual soundcard
* Mopidy - music player daemon, extensible with plugins
* Xvfb - headless X display server, needed by teamspeak
* x11vnc - to give remote connectivity to Xvfb, mostly for setup

Optional, for Google Play Music:

* Python
* gmusicapi (via pip)
* Moopidy-Gmusic (via pip)


