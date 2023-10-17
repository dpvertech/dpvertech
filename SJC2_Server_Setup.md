# SJC2 (Rap Lab) Server Setup

## Background

Brian Arebelo setup 2 linux servers in the SJC2 Raplab site.  This document goes through the various ways in which we are going to install ignition and get them working

## Overall Information

At SJC2 Rap Lab, we have 2 servers:

- 10.150.162.38 - rl-ignition01-r23-u24 - rl-ignition01 (axis)
- 10.150.162.39 - rl-ignition02-r23-u24 - rl-ignition02 (axis)

Username for both servers is `nvidian` - password can be found in lastpass. Reach out to David Folkner if you do not have access to this information in LastPass.

Jump Servers:

- 24.51.5.188
- 24.51.5.189

Username for both jump servers are your generic Nvidia username, and no password should be necessary once you use `nvinit`.

## Access

### SSH Tunneling to Servers

Nvidia has a pretty robust system for accessing production systems - Particularly RapLab.  <https://nvidia.sharepoint.com/sites/SAE-TME/SitePages/Accessing-the-SJC-2-RAPLAB.aspx#nvinit-method-nvidia-employees-only> is a great resource that was used to get access.  (Note you need an Nvidia account and to be on the VPN to access this).

#### Getting Access

First, you need to request access to the user group at SJC2 - <https://dlrequest/GroupID/Home/Index> with `RAPLabAccess`

#### Installing and using nvinit

The documentation provides decent instructions, but these are little clearer.  For either method I recommend going to:

<https://urm.nvidia.com/artifactory/swngc-ngcc-generic/nvinit/> instead of the exact version so you can pick the right version.  *Note: You have to use basic auth on these pages, just your nvidia id such as dfolkner and your nvidia password*.

##### Windows Instalation

Follow the same link in the given documentation.

1. Download the file
2. Extract the file using windows or 7zip to a local location
3. Place the nvinit in a folder that has the path - There are a million ways to do windows paths some options are:
   1. Copy the executable to a folder that is already in the path 
   2. Add a new folder to the path by editing the enviornement variables
   3. I use CMDER, so I just copy the file to `<cmder_file_path>/bin`
4. Check to make sure the executable is in your path by opening your command line of choice (I like cmder) but cmd or powershell can work too by running `where nvinit` (*with cmder you have linux commands so you can run*: `which nvinit` *like you can with linux below*)
5. run `nvinit --version` to make sure it runs

Windows typically has openssh-client already installed.  You shouldn't need openssh server.  But if for some reason you don't have ssh client (which should be just `ssh` in your command line) you can follow <https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse?tabs=powershell>

The `nvinit` also relies on the `ssh-agent service` which at least for me already was installed but was disabled.  You can enable this by the command line or just go to windows services and make it automatic and start it:

![SSH Agent Service Example](pictures/agent_service.png)

Now that this done you can actuall use `nvinit`

##### Linux Installation

`nvinit` is a tool developed internally by Nvidia that initiates the connection, certificates and what not to allow tunneling to the site.

Installing `nvinit` isn't great on windows yet, so a linux mint VM is preferable.  While on the VPN (See that section within your linux VM).  

To do this via the command line:

```bash
# Extract and move
cd Downloads/
tar -xzvf nvinit-2.1.8-linux-amd64.tgz
sudo cp nvinit /usr/bin/
sudo cp nvinit-ssh-helper /usr/bin/
sudo cp nvssh-webtunnel /usr/bin/

# Checks
which nvinit
nvinit --version
```

##### Use nvinit to get certs

Then to use `nvinit` do the following:

```bash
nvinit ssh -use-device-code
```

Then you will copy the URL go to your browser enter the code.  Log in with Nvidia.  Then you should get a green checkmark success.

#### Actually accessing the server

At this point you should have ssh access to the jump servers by doing `nvinit ssh <yournvidiausername>@<jumpserverIP>` such as `nvinit ssh dfolkner@24.51.5.188`.  If you can't do this, then the rest won't work.

Everything you want to do now is done with your ssh client on either linux or windows.  Most likely this is openssh.  You can do a LOT of cool things and you can do it a LOT of different interesting ways.  This does not cover everything you want to do, but it does give you some quick options for how to set this up.

##### Define your ssh config file

This is in my opinion the easiest way to actually to do everything.  This preconfigures some of the connections to make your life easier.  This file is generally located in your home folder `~/.ssh/config` for linux or `C:\Users\<USERNAME>\.ssh\config`

```conf
# Add the actual jump server SSH commands here
Host jump
  HostName 24.51.5.188
  # Replace with your username to go along with your nvinit
  User dfolkner

# Here is how you can quickly ssh into a server through the defined jump server using -W
Host primary_ignition
  HostName 10.150.162.38
  ProxyCommand ssh -W %h:%p jump
  User nvidian

  # This line opens the port 8088 on your local server when you are ssh'd in
  LocalForward 8088 localhost:8088
  LocalForward 8043 localhost:8043

# Similar to above, but maps port 8089 isntead of 8088
Host secondary_ignition
  HostName 10.150.162.39
  ProxyCommand ssh -W %h:%p jump
  User nvidian
  LocalForward 8089 localhost:8088
  LocalForward 8044 localhost:8043

# This gives you a shortcut to not actuall have a command line for any server but just open
# the ports to your local host.  You leave that command line open and you'll have access
# to http://localhost:8088 or 8089 as you want.  Close it by using CTRL+C

# TODO: Make these use the definitions above some how instead of redefining.
Host tunnel
  HostName 24.51.5.188
  User dfolkner
  LocalForward 8088 10.150.162.38:8088
  LocalForward 8089 10.150.162.39:8088
  LocalForward 8043 10.150.162.38:8043
  LocalForward 8044 10.150.162.39:8043
  # Sleep infinity is easy enough, but may not work.  Try tail -f if it doesn't work.
  RemoteCommand echo "Tunneling through Jump machine. CTRL+C to terminate connection"; sleep infinity
```

With this you can just SSH like normal using the keywords such as `primary_ignition` or `tunnel`

##### Setup a key instead of password access

It is best practice to setup a keystore or something similar to access the servers instead of using passworded access.  `nvinit` did an alternate version of this for the jump server, thus the reason you don't need to enter your password when accessing.  

To do this we are going to:
1. run `ssh-keygen` - This will make two files in your `.ssh` folder a private and public key.  Just press enter through most of the options.
2. Most tutorials have you use ssh-copy-id, but windows doesn't have this command often.
3. ssh into the server - `ssh primary_ignition` - use the password
4. Make sure that the `.ssh` exist or make it
5. `exit` - to leave ssh mode
6. `scp .ssh\id_rsa.pub primary_ignition:~\.ssh\authorized_keys` again using the known password
   1. If the file already exists, you'll need to manually add it to that file instead of overwriting it.

Now you can ssh to that sever without using a password which is actually more secure.  Cool thing here is the key is only on the end server, not the jump server at all.

##### Connect with VSCODE

One of the cool things about doing it this way with the config files and everything is that you can connect via VSCODE with the Remote-SSH Package.  This allows you to have a simple file editor on the server and a terminal built in to VSCODE if you'd like.  

##### OLD - SSH Into the servers

The best way is to use the above config and just run `ssh primary_ignition`

The easiest way to ssh into our actuall servers is to:

```bash
ssh dfolkner@24.51.5.188
ssh nvidian@10.150.162.38
```

You may need to press yes and enter the password, but you have SSH access at that point and can do what you need to from a backend perspective

##### OLD - Open ports through the jump server

On our servers, we'll have Ignition running which has a couple of different ports and webservers that we need to access.  The following will allow you to tunnel those ports open so you can access them on your local machine and/or VM.  This assumes that you are on the VPN still and the rest above is working.

The basic command to open map a port to your local machine is: `ssh -v -N -L 8088:10.150.162.38:8088 dfolkner@24.51.5.188` or more generically you can write this as `ssh -v -N -L localport:serverIP:serverPort Username@jumpip`.  Once you have done this, you can navigate to <http://localhost:8088> and you will be able see it as if you were local on the machine.  Here's how it works:

- `ssh username@jumpip` tells the system to ssh to the jump server just like normal
- `-v` tells the system to be verbose and give you LOTS of details
- `-N` tells the system to not actually connect to the jump server.  Since we don't actually want to run any commands ON the jump server, we just want to use it to forward ports.
- `-L` is the flag that you use to tell the jump server which ports to forward to you.  Multiple entries are possible.  You can do a BUNCH with this option but in general - `localport:serverIP:serverPort`
  - `localport` - this is the port you want to use on YOUR computer or VM.  This can be whatever you want.  Typically this would be where you access <http://localhost:localport>
  - `serverIP` - is just the server that you want to grab
  - `serverPort` - This is the webserver port.  For ignition this may be 8088, 8043, or even 8060 depending on what you are trying to do.

Here is an example of me doing this for both servers without SSL certs: `ssh -v -N -L 8088:10.150.162.38:8088 -L 8089:10.150.162.39:8088 dfolkner@24.51.5.188`.  Now I can access 01 at <http://localhost:8088> and 02 at <http://localhost:8089>

### SSH Only Access

NOTE: **THIS ONLY WORKS AT SJC2 AND DOESN'T ALLOW FOR IGNITION ACCESS**

Raplab is special and uses a tool called Axis.  Again Brian Arebelo commissioned Axis for us at Vertech.

Access to: <https://axis-nvidia.axisportal.io/apps> is needed

The servers should be listed there as

- rl-ignition01
- rl-ignition02

YOu login with the username: `nvidian`

Password can be found in the Lastpass Notes for both machines. Reach out to David Folkner if you do not have access to this information in LastPass.

### Tunneling

<https://nvidia.sharepoint.com/sites/SAE-TME/SitePages/Accessing-the-SJC-2-RAPLAB.aspx#nvinit-method-nvidia-employees-only>

## Server Setup

We have done very little to these servers.  We always run `sudo apt update` and `sudo apt upgrade -y` just to keep them current.

We also install the following packages:

- unzip
- net-tools (optional)

Added CDPATH to the `.bashrc`

## Installing Ignition

### Downloading Ignition

1. Go to <https://inductiveautomation.com/downloads/> and navigate to "other versions"
2. Right click and copy the link to the Linux ZIP that you would like to use to get the url.
3. On the local server run: `wget -q --ca-certificate=/etc/ssl/certs/ca-certificates.crt --referer https://inductiveautomation.com/* -O "ignition-install.zip" "<URL>"`
4. Then copy the checksum and run `echo "<CHECK SUM>" "ignition-install.zip" | sha256sum -c -`

### Inistalling the Service

Use the readme in the zip here: <https://docs.inductiveautomation.com/display/DOC81/ZIP+File+Installations>

1. run `unzip -p ignition-isntall.zip README > README`
2. `vi or nano README` Then follow those instructions
3. run `sudo unzip ignition-install.zip -d /usr/local/bin/ignition`

Sample Readme:

```md
Installing and running Ignition from zip

* New installation from zip *
1) Create a folder in which to unzip Ignition. A common choice is /usr/local/bin/ignition (note that you must have superuser access or use sudo to create this folder).
`mkdir /usr/local/bin/ignition`
2) Set the owner of the newly created ignition folder. Note that you may need to have superuser access or use sudo to run this command.
`chown myusername /usr/local/bin/ignition`
3) Unzip into the ignition folder:
`unzip Ignition-linux-64-X.X.X.zip -d /usr/local/bin/ignition`
4) Navigate to the newly unzipped ignition folder and set the executable permission on all .sh files in the folder.
`cd /usr/local/bin/ignition`
`chmod +x *.sh`
5) (Optional) To install as a system service that auto-starts Ignition on system boot, run:
`./ignition.sh install`
6) Start the Ignition Gateway. If the ignition folder is owned by root, you must have superuser access or use sudo to run:
`./ignition.sh start`
7) Open a web browser and navigate to http://localhost:8088. You can now start the Gateway commissioning process.
If any errors occur during startup, they will be logged to logs/wrapper.log

You can also add the unzipped folder to your system PATH by adding the folder path to a file such as .profile or .bashrc. This allows you to start programs like the Gateway Control Utility from the command line without specifying a complete path to the unzipped folder.

* Starting/stopping/restarting Ignition *
To start Ignition, run:
`./ignition.sh start`

To stop Ignition, run:
`./ignition.sh stop`

To restart Ignition, run:
`./ignition.sh restart`

* Upgrading existing installation *
Note: Maker Edition installations should not be updated using a ZIP installer. Instead, please use the latest graphical install wizard which now supports both installations and updates to all editions, including those originally installed using a ZIP installer. Note that this installer can also be run in headless>If you have Ignition running out of a previous unzipped folder, and you would like to upgrade, follow these steps:
1) Take a Gateway backup from the running system and store in a safe location.
2) Stop Ignition. Run:
`./ignition.sh stop`
3) If Ignition was installed as a service, remove the service:
 `./ignition.sh remove`
4) Delete both the "lib" and "webserver/webapps" folders.
5) Unzip the new zip into the current folder and allow files to be overwritten EXCEPT for data/ignition.conf.
6) Set the executable permission on all .sh files in the unzipped folder:
`chmod +x *.sh`
7) Unzip the new runtime:
`./ignition.sh checkruntimes`
8) If this is a major version upgrade (8.0 to 8.1) then the data/ignition.conf file may have changed. If this is a minor version upgrade (8.0.1 to 8.0.2) then skip to step 8.
   To upgrade the ignition.conf file for a major version upgrade, run the following from command line in the ignition folder:
`./ignition.sh runupgrader`
9) (Optional) To install as a system service that auto-starts Ignition on system boot, run:
`./ignition.sh install`
10) Start Ignition. Run:
`./ignition.sh start`

* Debian-based systems: running from installer *
If you installed Ignition from the installer .run file, and you are running a Debian-based system (such as Ubuntu or Mint), the installer created a service for you. The service will start Ignition automatically when the machine is started. To start the Ignition service, run:
`./ignition.sh start`
```
