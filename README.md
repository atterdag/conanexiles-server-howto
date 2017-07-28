# Installing Conan Exiles on Debian Stretch                                          #
There are several articles that helps you with how to install a Conan Exiles Dedicated Server on Linux. But none of them seems to take security into account, so I decided to write my own.

## Source material used
* [Installing Wine on Debian Jessie](https://wiki.winehq.org/Debian)
* [Running a Conan Exiles Server on Ubuntu Linux using WINE By Runningman](http://steamcommunity.com/sharedfiles/filedetails/?id=869441506)
* [Conan Exiles How To Create Your Own Dedicated Server (With Admin Commands)](http://www.gamerevolution.com/faq/conan-exiles/how-to-create-your-own-dedicated-server-with-admin-commands-129183)
* [Conan Exiles dedicated server configuration](https://gist.github.com/Dids/9564a693bb5b40cc0773cf8a5f07b4c1)

## Prerequisites
* A server with sufficient resources for your configuration
* Debian Stretch amd64 installation
* At least 4 GB available free space on /home partition, and 1G on /opt
* Set up a firewall - I recommend UFW, as it's easy to install, and manage.
* Install some sort of security policy extension, such as SELinux or AppArmor. I prefer SELinux because of its ability to automatically generate new policies. But whatever is preferred.

If you want some tips on how to set up SELinux, or UFW on Debian, then check out my [Typical Debian Stuff (Updated for Jessie) blog post](http://valdemar.lemche.net/2014/03/typical-debian-stuff.html)

## Preparing the OS                                                           #
If using SELinux, set it in permissive mode.

```
$ sudo setenforce 0
```
 
Install X Virtual Framebuffer, screen, and a HTTPS extension to APT, as well as ensuring that sudo, and sqlite3 is installed.

```
$ sudo apt-get install sudo xvfb screen apt-transport-https sqlite3 libasound2-plugins:i386
```

## Installing Wine                                                            #
Allow 32 bit packages:

```
$ sudo dpkg --add-architecture i386
```

Download, and add key used to sign wine packages.

```
$ wget https://dl.winehq.org/wine-builds/Release.key && sudo apt-key add Release.key
```

Add the repository to **/etc/apt/sources.list.d/**

```
$ cat << EOF | sudo tee /etc/apt/sources.list.d/winehq.list
deb https://dl.winehq.org/wine-builds/debian/ stretch main
EOF
```

Update package list

```
$ sudo apt-get update
```

Install Wine packages

```
$ sudo apt-get install --install-recommends wine-staging
```

## Installing Steam                                                           #
Install steamcmd

```
$ sudo apt-get install --install-recommends steamcmd
```

## Installing Conan Exiles as runtime user                                    #
Create steam group

```
$ sudo groupadd -r steam
```

Create runtime user

```
$ sudo useradd -g steam -m -s /bin/false steam
```

Become runtime user

```
$ sudo su - steam -s /bin/bash
```

First initialize your Wine environmnet
```
$ /opt/wine-staging/bin/wineboot
```

Download Conan Exiles

```
$ steamcmd +@sSteamCmdForcePlatformType windows +force_install_dir /home/steam/conanexiles +login anonymous +app_update 443030 validate +quit
```

Test that you can start Conan Exiles server

```
$ xvfb-run --auto-servernum --server-args='-screen 0 640x480x24:32' /opt/wine-staging/bin/wine64 /home/steam/conanexiles/ConanSandboxServer.exe -log
```

After it starts up, then just kill it with Ctrl+C

## Configuring Conan Exiles                                                   #
Still as runtime user, create the configuration files. Change the values below to what you want, such as passwords, and server name.

NB: With the current release I'm not really able to tell the server to listen to other ports than the new default ports 7778/udp, and 14001/udp.

```
$ cat > ~/conanexiles/ConanSandbox/Saved/Config/WindowsServer/Engine.ini << EOF
[OnlineSubsystemSteam]
ServerName="atterdag's super test server"
ServerPassword=secret
# Default server port is 7778
#Port=7778 # Don't set it here, and set it the start script
# Default query port is 27015
#QueryPort=14001 # Don't set it here, and set it the start script
EOF

$ cat > ~/conanexiles/ConanSandbox/Saved/Config/WindowsServer/Game.ini << EOF
[ConanSandbox]
# set to the username of your runtime user
UserID=steam

[/script/engine.gamesession]
# Raise this number to allow more concurrent players - duh!
#MaxPlayers=10 # Don't set it here, and set it the start script
EOF

$ cat > ~/conanexiles/ConanSandbox/Saved/Config/WindowsServer/ServerSettings.ini << EOF
[ServerSettings]
# Its a good idea to set a strong password here
AdminPassword=SuperSecret
# 0=no nudity, 1=semi nudity, 2=full nudity
MaxNudity=2
# To disallow PVP then set to False
PVPEnabled=True
# 0=none, 1=Purist, 2=Relaxed, 3=Hardcore, 4=roleplaying, 5=experimental
ServerCommunity=5
IsBattlEyeEnabled=True
EOF
```

Its also in ServerSettings.ini that you set all the settings that you can do in-game as Administrator. And its kinda easier to do it from there.

If you'r running a LAN Server, then its a good idea to raise the values as below

**Engine.ini**

```
[OnlineSubsystemSteam]
AsyncTaskTimeout=60

[/script/onlinesubsystemutils.ipnetdriver]
MaxClientRate=100000
MaxInternetClientRate=100000 
```

**Game.ini**

```
[/script/engine.gamenetworkmanager]
TotalNetBandwidth=4000000
MaxDynamicBandwidth=100000
MinDynamicBandwidth=40000
```

Log off the runtime user

```
$ logout
```

## Create SysV script to manage Conan Exiles                                  #
systemd only support predefined actions such as start/stop/restart etc. So to have actions such as backup, update, connect to screen, then we need to make a traditional SysV script

```
$ sudo wget -O /etc/init.d/conanexiles https://raw.githubusercontent.com/atterdag/conanexiles-server-howto/master/conanexiles.init
```

Make the script executable

```
$ sudo chmod +x /etc/init.d/conanexiles
```

Tell OS to run script at boot

```
$ sudo update-rc.d conanexiles defaults
```

The SysV script supports the following actions
* start - Start the server
* stop - Stop the server
* restart - Restart the server
* status - Check if server is running or not
* connect - Connect to screen session
* update - Update game files
* backup - Backup server configuration, and database
* restore - Restore server from archive file
* checkgamedb - Verify game databases integrety

But for now we'll just start it

```
$ sudo /etc/init.d/conanexiles start
```

## Configure UFW to allow incoming traffic to Conan Exiles                    #
Create UFW application profile for Conan Exiles default ports

```
$ cat << EOF | sudo tee /etc/ufw/applications.d/conanexiles
[conanexiles]
title=Conan Exiles Server
description=Conan Exiles dedicated server default ports
ports=7778/udp|7779/udp|14001/udp
categories=Game;
reference=[https://github.com/atterdag/conanexiles-server-howto]
EOF
```

Tell UFW to load the profile

```
$ sudo ufw allow conanexiles
```

If you are setting your own Netfilter script, then I'm sure you can add these ports to it yourself.

## Wrapping up the OS                                                         #
Enforce SELinux polices again

```
$ sudo setenforce 1
```

Clean apt cache

```
$ sudo apt-get clean
```
