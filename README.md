# MojoBot - Teamspeak bot + Mopidy remote control written in Mojolicious #

What you need to get this working:

* Pulseaudio - to provide the "link" between mopidy's output and teamspeak/s input, as a virtual soundcard
* Mopidy - music player daemon, extensible with plugins
* Xvfb - headless X display server, needed by teamspeak
* x11vnc - to give remote connectivity to Xvfb

For mopidy:

* Python
* gmusicapi (via pip)
* Moopidy-Gmusic (via pip)

For the web frontent:

* Perl
* mojolicious (from CPAN or package)