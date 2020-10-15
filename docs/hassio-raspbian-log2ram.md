---
layout: default
title: Installing hass.io (Home Assistant + Supervisor) on Raspbian with log2ram
nav_order: 2
---

# Installing hass.io (Home Assistant + Supervisor) on Raspbian with log2ram



Intro

## Installing and Configuring the Raspberry-Pi-OS

### Installing Raspberry-Pi-OS

The Raspberry Pi foundation documentation does a good job of describing how to flash an image to the SD card and getting your Pi up-and-running. You can find the official guide [here](https://www.raspberrypi.org/documentation/installation/installing-images/). You'll want to use the "Raspberry Pi OS (32-bit) Lite" image for this application [link](https://www.raspberrypi.org/downloads/raspberry-pi-os/). 

I also recommend enabling ssh on your Raspberry Pi before booting it for the first time. You can do this by creating an empty file in the `BOOT`folder on your SD card. 

- Windows Users: Create a .txt file in the `BOOT` directory on your SD card and rename it to `ssh` <u>without an extension</u>. You may have to change your folder settings to show extensions on known filetypes [link](https://www.howtohaven.com/system/show-file-extensions-in-windows-explorer.shtml).
- Linux Users: You can either do something similar to what I mentioned for Windows users, or if you're handy with the terminal you can navigate to the `BOOT`directory and type: `touch ssh` 

The Raspberry Pi foundation also has a tutorial on configuring the Raspberry Pi for headless operation [link](https://www.raspberrypi.org/documentation/configuration/wireless/headless.md).

### Configuring Raspberry-Pi-OS

Once you've connected to the Pi (the default login information is: U:pi P:raspberry), run `sudo raspi-config` to bring up the configuration window.

1. Select the first option (Change User Password) and set a new password.
2. Select the fourth option (Localisation Options) and select `Change Time Zone`. Home Assistant needs to know your current time zone so that logs and other things behind-the-scenes work correctly.
3. Feel free to adjust other settings as necessary. Once you're finished, select `Finish` and allow the Pi to reboot. 

Once the Pi reboots, enter the following command to update the OS. 

```
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
```

## Installing Docker

Installing Docker is very straightforward. Enter the following commands in the order shown:

### Install Docker

```
curl -sSL https://get.docker.com | sh
```

### Add the  "pi" User to the "docker" group

This command is necessary for the "pi" user to interface with docker containers. 

```
sudo usermod -aG docker pi
```

After executing this command, **reboot the Pi**.

### Test the Docker installation

```
docker run hello-world
```

If this command shows you a "hello world" message, then docker should be working correctly. 

**Install Docker dependencies**

```
sudo apt-get install -y libffi-dev libssl-dev
sudo apt-get install -y python3 python3-pip
```

These dependencies are only required for Docker to run. I also use docker-compose to manage other containers outside of hass.io, so executing the command below will install docker-compose.

```
sudo pip3 -v install docker-compose
```

Docker should now be ready! 

## Installing the Supervisor and Dependencies

Installing the hass.io supervisor outside of the installer provided by the Home Assistant team is **unsupported**. All this means is that if you run into any issues, the Home Assistant developers won't be able to (or want to) help. 

The Home Assistant development team provides an install script for setting up the supervisor (hass.io) on a system outside of the hass.io image. The script is hosted on the project site [here](https://github.com/home-assistant/supervised-installer). 

A few dependencies need to be installed and a service needs to be disabled before you can run the script. Execute the commands below to get things set up.

```
sudo -i

apt-get install -y software-properties-common apparmor-utils apt-transport-https avahi-daemon ca-certificates curl dbus jq network-manager

systemctl disable ModemManager

systemctl stop ModemManager
```

Once everything is installed and configured, issue the command below, substituting `MY_MACHINE` with your Raspberry Pi model (supported machine types shown below).

```
curl -sL https://raw.githubusercontent.com/home-assistant/supervised-installer/master/installer.sh | bash -s -- -m MY_MACHINE
```

```
raspberrypi
raspberrypi2
raspberrypi3
raspberrypi4
```

**Note: The script supports 64-bit installations, but Raspberry-Pi-OS is a 32-bit OS!**

Home Assistant (hass.io) should be available a few minutes after the script is finished. Use the command below to watch the Docker container start up. You can exit the log monitor by pressing Ctrl + C.

```
docker logs hassio_supervisor -f
```

## Installing and Configuring log2ram

log2ram creates a mount point in RAM where applications can use to write log data instead of writing directly to the SD card. Data logged in RAM is then periodically copied to the SD card on a predetermined schedule. The log2ram [project](https://github.com/azlux/log2ram) is hosted on GitHub and has a short list of commands needed to get it installed. Once you've rebooted your Pi, edit the log2ram configuration file located at `etc/log2ram.conf`

I changed the default size and log location to what's shown below.

```
SIZE=128M
...
PATH_DISK="/var/log;/usr/share/hassio/homeassistant/logs"
```

If you ran the hass.io supervisor installation script with the default path settings, this path should also be valid for your installation. 

## Configuring hass.io to use log2ram

We need to modify a few configuration files and create a symlink before hass.io is ready to use. 

Navigate to `/usr/share/hassio/homeassistant` and create a new directory for your logs (`sudo` is required for all of these commands). 

``` 
sudo mkdir logs
```

Add the text below to your `configuration.yaml` file using a text editor or the web interface. This command tells Home Assistant to write the SQLite database to the `logs/` folder in the Home Assistant configuration directory. **Note: This path is meant to be relative! Remember, Home Assistant is running in a Docker!**

```yaml
recorder:
  db_url: sqlite:///logs/home-assistant_v2.db
```

Once you've saved and closed the file, navigate to `Configuration > General > Server Controls` and **stop** the server. Be sure to wait a minute or two for Home Assistant to finish writing to the database and log files.

Move the database and log files into the `logs/` directory.

```
cd /usr/share/hassio/homeassistant/
sudo mv home-assistant.log logs/
sudo mv home-assistant_v2.db logs/
```

Create a symlink to the Home Assistant log file using the command below.

```
cd /usr/share/hassio/homeassistant/
sudo ln -s logs/home-assistant.log home-assistant.log
```

Start log2ram to have it initialize the directories. 

```
sudo log2ram start
```

Executing this command should've created a new folder in the `homeassistant/` directory called `hdd.logs`. This directory is where log2ram will write the logs as per the schedule you configured earlier. You should have also seen the `rsync` output detect and write the two log files we just moved along with `log2ram.log`.

```
sent 4,736,778 bytes  received 46,093 bytes  9,565,742.00 bytes/sec
total size is 18,661,461  speedup is 3.90
building file list ... done
./
home-assistant.log
home-assistant_v2.db
log2ram.log
```

## Conclusion

That's it! Once you reboot your Pi, your logs should now be written to RAM and periodically backed up to the SD card! 