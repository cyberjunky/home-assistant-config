# My Home Assistant Setup

This repository holds the setup (hardware) and configuration of my Home Automation using [Home Assistant](https://home-assistant.io/). After a long time experimenting and looking at other peoples [users configs](https://github.com/search?o=desc&q=topic%3Ahome-assistant-config&s=stars&type=Repositories) and browsing through the [forum](https://community.home-assistant.io/latest) I think I have figured out the layout I'm most comfortable with.

I will also use this repository to create issues holding ideas and items to fix, now they are scattered over paper notes, text files and Google Keep, it's a chaos.

I just started to set this up and will be expanded over the coming period.
Please :star2: my repository if you like it and want see the progress and the experiments.

Also have a look at my [Custom Components](https://github.com/cyberjunky/home-assistant-custom-components)

## Generic Setup

### Server

<img src="https://github.com/cyberjunky/home-assistant-config/blob/master/docs/server/fc8-evo-featured.jpg" width="150"> 

My setup is running on a custom build server using passive cooling running Ubuntu 18.04.1 (bionic) 64 bits.
Nowadays I would buy a Intel NUC I guess, but the hardware was there already and is performing well.

You can find detailed server build instructions [here](https://github.com/cyberjunky/home-assistant-config/blob/master/docs/server/INSTALL.md).

### Home Assistant

<img src="https://github.com/cyberjunky/home-assistant-config/blob/master/docs/server/hass-icon.jpg" width="50"> 

I'm using the Hass.io image running inside docker, simply because it's easy to maintain so I can focus on all the other stuff.

I installed it like described on [this page](https://www.home-assistant.io/hassio/installation/#alternative-install-on-generic-linux-server).

Here the commands I used:

```
$ sudo add-apt-repository universe
$ sudo apt-get update
$ sudo apt-get install -y apparmor-utils apt-transport-https avahi-daemon ca-certificates curl dbus jq network-manager socat software-properties-common
$ mkdir hassio
$ wget https://raw.githubusercontent.com/home-assistant/hassio-build/master/install/hassio_install
$ chmod +x hassio_install
$ sudo ./hassio_install
[Warning] No NetworkManager support on Host.
[Warning] Create DNS settings for docker to avoid systemd bug!
[Info] Restart docker and wait 30sec
[Info] Install supervisor docker
[Info] Install supervisor startup scripts
Created symlink /etc/systemd/system/multi-user.target.wants/hassio-supervisor.service → /etc/systemd/system/hassio-supervisor.service.
[Info] Install AppArmor scripts
Created symlink /etc/systemd/system/multi-user.target.wants/hassio-apparmor.service → /etc/systemd/system/hassio-apparmor.service.
[Info] Run Hass.io
```

```
$ docker container list | grep hassio_
fbb2bb464414        homeassistant/amd64-hassio-supervisor    "python3 -m hassio"      2 weeks ago         Up 5 days                                                                                                        hassio_supervisor
```
First startup can take some time.

The Hass.io GUI is available from Home Assistant's menu at: http://[IP server]:8123

*I will include more documentation soon!*

## Screenshots

## Resources
