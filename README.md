## DIY HOME SERVER - PROXMOX - UPS

When building a home lab server, you’re mostly going to use some kind of Uninterruptible Power Supply (UPS).

It’s important for your home servers to always know the status of the UPS providing the power. This makes it possible to gracefully shutdown all systems before the UPS batteries get drained.

**Cyberpower CP1500 AVR UPS**

In most cases, home users are running one or multiple Proxmox bare metal servers, each running several VMs and containers. Each virtualization host can shutdown all of its own clients during a normal shutdown sequence.  But therefore each host has to be aware of the UPS status.

Most UPS have one port to connect them to computers or servers for management purposes. On the budget friendly devices, you’ll find a USB or serial connector. The more expensive devices often have an ethernet port.

My Cyberpower CP1500 AVR UPS is just a cheap and simple 650 Watt/1.4 kVA device. The UPS has one USB port to communicate with connected devices.

> Tip :
Before buying a certain UPS, always first check replacement batteries prices and availability!
I especially chose this UPS because replacement batteries are widely available, affordable and don’t have any exotic dimensions.
> 

## DIY HOME SERVER - PROXMOX - NUT

We’ll use the Open Source Network UPS Tools (NUT) to communicate with the UPS.

Because there’s only one (USB) port, this means that only one computer or server can communicate with the UPS directly. This brings us to the following topologies :

### “Simple” configuration :

One UPS, one computer or server. This is also known as a “Standalone” configuration.

This is the configuration that most users will use. You need at least a driver, upsd, and upsmon running.

### “Advanced” configuration :

One UPS, multiple computers or servers. Only one of them can actually talk to the UPS directly. That’s where the network comes in. The Master system runs the driver, upsd, and upsmon in master mode. The Slave systems only run upsmon in slave mode.

This is useful when you have a very large UPS that’s capable of running multiple systems simultaneously. There is no longer the need to buy a bunch of individual UPS or “sharing” hardware, since this software will handle the sharing for you.

In our example, we’ll setup a “simple” configuration. The UPS USB port is connected directly to an USB port on the Proxmox server.

**Shutdown timeline (simple configuration) :**

- Main power failure occurs :
    - UPS device switches power to battery
    - UPS device notifies server with a “On battery” event message
    - UPS battery is getting close to depletion :
        - UPS device notifies server with a “Battery low” event message
        - Server waits the set “Final delay” time
        - Server starts his shutdown procedure :
            - Sets the “Kill power” flag
            - Ends all running processes
            - Unmounts all file systems
            - Remounts file systems as read-only
            - Looks for the “Kill power” flag
            - Issues the “Kill power” command to the UPS device
            - Halts the system, but doesn’t power off
            - UPS device receives the “Kill power” command from the server :
                - UPS waits for the “Shutdown delay” time to pass
                - This is to give all systems enough time to properly shut down
                - UPS device cuts power on all outlets
                - All connected systems lose power
- Main power supply has been restored :
    - UPS device starts to reload its battery
    - UPS device waits for the “Startup delay” time to pass | This is to reload the battery to a safe minimum level
    - UPS device restores power on all outlets
    - All connected systems start up again

If you want to set up an advanced configuration, please consult [this excellent post](https://roll.urown.net/server/server-ups.html).

NUT server and NUT client directly on to our Proxmox PVE.
The server will communicate with the UPS over the USB connection and serve the data to all listening clients.
The client will listen to the server and take appropriate actions depending on the UPS status.
NUT client and an Apache webserver on a dedicated LXC container. This client will also listen to the server and present the UPS statistics in a graphical user interface.
We use a separate LXC container because, for security reasons, we don NOT want to install a webserver directly onto our Proxmox PVE.
NUT is a pain to configure. But it’s well documented and worth the pain.

DIY HOME SERVER - PROXMOX - NUT Installation
In Proxmox, select your PVE node → Shell.
Or use your favorite remote connection tool to connect to your proxmox server. We recommend Putty for Windows.

List all USB ports :
lsusb
Note the Bus and Device of your UPS device.

List the USB port details :
lsusb -v -s [bus]:[device]
In my case :
lsusb -v -s 5:50

Tip : If the above does not work, replace the USB cable and reboot the system.

Install NUT :

apt install nut -y
Probe the NUT devices :

nut-scanner -U
Take note of these values.

a. /etc/nut/nut.conf
Backup /etc/nut/nut.conf :
cp /etc/nut/nut.conf /etc/nut/nut.example.conf

Remove /etc/nut/nut.conf :
rm /etc/nut/nut.conf

Edit /etc/nut/nut.conf :
nano /etc/nut/nut.conf

Add :

MODE=netserver

b. /etc/nut/ups.conf
Backup /etc/nut/ups.conf :
cp /etc/nut/ups.conf /etc/nut/ups.example.conf

Remove /etc/nut/ups.conf :
rm /etc/nut/ups.conf

Edit /etc/nut/ups.conf :
nano /etc/nut/ups.conf

Add :

pollinterval = 15
maxretry = 3

offdelay = 120
ondelay = 240

[apc]
driver = usbhid-ups
port = auto
vendorid = 0764

Use the values returned by the nut-scanner command above.

After saving the file, you can test it by entering the command :
upsdrvctl start

You should get a return like :
Network UPS Tools - UPS driver controller 2.8.0
Network UPS Tools - Generic HID driver 0.47 (2.8.0)
USB communication driver (libusb 1.0) 0.43

Broadcast message from root@pve (somewhere) (Sat Aug  3 10:41:06 2024):

Communications with UPS apc@localhost established

Please check the Network UPS Tools – Hardware Compatibility List to identify the correct driver suited for your UPS.

c. /etc/nut/upsd.conf
Backup /etc/nut/upsd.conf :
cp /etc/nut/upsd.conf /etc/nut/upsd.example.conf

Remove /etc/nut/upsd.conf :
rm /etc/nut/upsd.conf

Edit /etc/nut/upsd.conf :
nano /etc/nut/upsd.conf

Add :

LISTEN 0.0.0.0 3493
LISTEN :: 3493
By using 0.0.0.0 instead of 127.0.0.1 we instruct the server to respond to requests from all networks.

d. /etc/nut/upsd.users
Backup /etc/nut/upsd.users :
cp /etc/nut/upsd.users /etc/nut/upsd.example.users

Remove /etc/nut/upsd.users :
rm /etc/nut/upsd.users

Edit /etc/nut/upsd.users :
nano /etc/nut/upsd.users

Add :

[upsadmin]

# Administrative user

password = ********

# Allow changing values of certain variables in the UPS.

actions = SET

# Allow setting the "Forced Shutdown" flag in the UPS.

actions = FSD

# Allow all instant commands

instcmds = ALL
upsmon master

[upsuser]

# Normal user

password = ********
upsmon slave
Replace ******** with your own password(s).

e. /etc/nut/upsmon.conf
Backup /etc/nut/upsmon.conf :
cp /etc/nut/upsmon.conf /etc/nut/upsmon.example.conf

Remove /etc/nut/upsmon.conf
rm /etc/nut/upsmon.conf

Edit /etc/nut/upsmon.conf :
nano /etc/nut/upsmon.conf

Add :

RUN_AS_USER root
MONITOR apc@localhost 1 upsadmin ******* master

MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h"
NOTIFYCMD /usr/sbin/upssched
POLLFREQ 4
POLLFREQALERT 2
HOSTSYNC 15
DEADTIME 24
MAXAGE 24
POWERDOWNFLAG /etc/killpower

NOTIFYMSG ONLINE "UPS %s on line power"
NOTIFYMSG ONBATT "UPS %s on battery"
NOTIFYMSG LOWBATT "UPS %s battary is low"
NOTIFYMSG FSD "UPS %s: forced shutdown in progress"
NOTIFYMSG COMMOK "Communications with UPS %s established"
NOTIFYMSG COMMBAD "Communications with UPS %s lost"
NOTIFYMSG SHUTDOWN "Auto logout and shutdown proceeding"
NOTIFYMSG REPLBATT "UPS %s battery needs to be replaced"
NOTIFYMSG NOCOMM "UPS %s is unavailable"
NOTIFYMSG NOPARENT "upsmon parent process died - shutdown impossible"

NOTIFYFLAG ONLINE   SYSLOG+WALL+EXEC
NOTIFYFLAG ONBATT   SYSLOG+WALL+EXEC
NOTIFYFLAG LOWBATT  SYSLOG+WALL+EXEC
NOTIFYFLAG FSD      SYSLOG+WALL+EXEC
NOTIFYFLAG COMMOK   SYSLOG+WALL+EXEC
NOTIFYFLAG COMMBAD  SYSLOG+WALL+EXEC
NOTIFYFLAG SHUTDOWN SYSLOG+WALL+EXEC
NOTIFYFLAG REPLBATT SYSLOG+WALL
NOTIFYFLAG NOCOMM   SYSLOG+WALL+EXEC
NOTIFYFLAG NOPARENT SYSLOG+WALL

RBWARNTIME 43200
NOCOMMWARNTIME 600

FINALDELAY 5
Replace ******** with your own password(s).

Tip :
If you frequently loose the USB connection (data becomes stale) between your server and the UPS, then change the values for :

POLLFREQ
POLLFREQALERT (= POLLFREQ / 2)
DEADTIME (= POLLFREQ * 6)
MAXAGE (= POLLFREQ * 6)

f. /etc/nut/upssched.conf
Backup /etc/nut/upssched.conf :
cp /etc/nut/upssched.conf /etc/nut/upssched.example.conf

Remove /etc/nut/upssched.conf
rm /etc/nut/upssched.conf

Edit /etc/nut/upssched.conf :
nano /etc/nut/upssched.conf

Add :

CMDSCRIPT /etc/nut/upssched-cmd
PIPEFN /etc/nut/upssched.pipe
LOCKFN /etc/nut/upssched.lock

AT ONBATT * START-TIMER onbatt 30
AT ONLINE * CANCEL-TIMER onbatt online

# AT ONBATT * START-TIMER earlyshutdown 30

AT LOWBATT * EXECUTE onbatt
AT COMMBAD * START-TIMER commbad 30
AT COMMOK * CANCEL-TIMER commbad commok
AT NOCOMM * EXECUTE commbad
AT SHUTDOWN * EXECUTE powerdown
AT SHUTDOWN * EXECUTE powerdown

A tip from TechnoTim :
Be sure PIPEFN and LOCKFN point to a folder that esists, I’ve seen it point to /etc/nut/upssched/ instead of /etc/nut/.
If it does, create the folder or update these variables. mkdir /etc/nut/upssched/

mkdir /etc/nut/upssched/

g. /etc/nut/upssched-cmd
(If exists) Backup /etc/nut/upssched-cmd :
cp /etc/nut/upssched-cmd /etc/nut/upssched-cmd.example

Remove /etc/nut/upssched-cmd

Edit /etc/nut/upssched-cmd :
nano /etc/nut/upssched-cmd

Add :

#!/bin/sh
case $1 in
onbatt)
logger -t upssched-cmd "UPS running on battery"
;;
#  earlyshutdown)
#     logger -t upssched-cmd "UPS on battery too long, early shutdown"
#     /usr/sbin/upsmon -c fsd
#     ;;
shutdowncritical)
logger -t upssched-cmd "UPS on battery critical, forced shutdown"
/usr/sbin/upsmon -c fsd
;;
upsgone)
logger -t upssched-cmd "UPS has been gone too long, can't reach"
;;
*)
logger -t upssched-cmd "Unrecognized command: $1"
;;
esac
Then :

chmod +x /etc/nut/upssched-cmd

Now reboot your Proxmox server. Or enter the following command sequence to restart the services :

service nut-server restart
service nut-client restart
systemctl restart nut-monitor
upsdrvctl stop
upsdrvctl start

Test your configuration by running the command :
upsc apc@localhost

You should see the UPS replying its data.

Tip : If the above does not work, replace the USB cable and reboot the system once again.

At the moment, the system is configured to shut down when the UPS runtime or battery charge values fall below their values. By default, those values are way too low, so we will set this parameter to a higher value to keep enough reserves in the UPS. You can list the editable values :
upsrw apc@localhost

We will set the parameter for minimum runtime to 10 minutes and the minimum charge to 50% :
upsrw -s battery.runtime.low=600 apc@localhost
upsrw -s battery.charge.low=50 apc@localhost
These commands will ask for a username and password. Be sure to use the upsadmin account and password specified in /etc/nut/upsmon.conf and /etc/nut/upsd.users. New values will be visible after restart.

DIY HOME SERVER - PROXMOX - NUT Monitoring
a. Create LXC container
Let’s create a new LXC container and install nut-client to monitor the UPS.
In Proxmox, select local (storage) → CT Templates → Templates.

In the dropdown, select the latest Ubuntu LTS (.04) template and click the Download button.

Click the Create CT button to create a new LXC.

On the General tab, specify the container name an set the password for the root user.

Click Next

On the Template tab, select the latest Ubuntu CT Template.

Click Next.

On the Root Disk tab, set a 2 GB Disk size.

Click Next.

On the CPU tab, there’s nothing to change.

Click Next.

On the Memory tab, there’s nothing to change.

Click Next.

On the Network tab, set the static IP address and specify the Gateway. (DHCP is also acceptable, but reserve the IP in your router)

Click Next.

On the DNS tab, there’s nothing to change.

Click Next.

On the Confirm tab, check the Start after created option.

Click Finish.

The container will be installed.

Select the container and click Options → Start at boot → Edit.

Set the Start at boot option and click OK.

In Proxmox, select the newly created LXC container and open the Console.

Update and upgrade the system :
apt update && apt upgrade -y

Install openssh-server to allow remote access :
apt install openssh-server -y

Configure the openssh-server :
nano /etc/ssh/sshd_config

Uncomment or add :

PasswordAuthentication yes
PermitRootLogin yes

Save the file.

Restart openssh-server :
systemctl restart ssh

Install an Apache webserver and nut-cgi :
apt install apache2 nut-cgi -y

Install NUT client :
apt install nut-client -y

b. /etc/nut/nut.conf

Backup /etc/nut/nut.conf :
cp /etc/nut/nut.conf /etc/nut/nut.example.conf

Remove /etc/nut/nut.conf :
rm /etc/nut/nut.conf

Edit /etc/nut/nut.conf :
nano /etc/nut/nut.conf

Add :
MODE=netclient

c. /etc/nut/hosts.conf
Backup /etc/nut/hosts.conf :
cp /etc/nut/hosts.conf /etc/nut/hosts.example.conf

Remove /etc/nut/hosts.conf :
rm /etc/nut/hosts.conf

Edit /etc/nut/hosts.conf :
nano /etc/nut/hosts.conf

Add :

MONITOR [apc@xxx.xxx.xxx.xxx](mailto:apc@xxx.xxx.xxx.xxx) "UPS Name"
Replace [xxx.xxx.xxx.xxx](http://xxx.xxx.xxx.xxx/) with the IP address of your Proxmox (NUT) server.

d. /etc/nut/upsset.conf
Backup /etc/nut/upsset.conf :
cp /etc/nut/upsset.conf /etc/nut/upsset.example.conf

Remove /etc/nut/upsset.conf :
rm /etc/nut/upsset.conf

Edit /etc/nut/upsset.conf :
nano /etc/nut/upsset.conf

Add :
I_HAVE_SECURED_MY_CGI_DIRECTORY

e. /etc/nut/upsmon.conf
Backup /etc/nut/upsmon.conf :
cp /etc/nut/upsmon.conf /etc/nut/upsmon.example.conf

Remove /etc/nut/upsmon.conf :
rm /etc/nut/upsmon.conf

Edit /etc/nut/upsmon.conf :
nano /etc/nut/upsmon.conf

Add :

RUN_AS_USER root
MONITOR [apc@xxx.xxx.xxx.xxx](mailto:apc@xxx.xxx.xxx.xxx) 1 upsuser ******* slave
Replace [xxx.xxx.xxx.xxx](http://xxx.xxx.xxx.xxx/) with the IP address of your Proxmox (NUT) server.
Replace ******** with your own password(s).

Enter the command :
sudo a2enmod cgi

Then restart the webserver :
sudo systemctl restart apache2

Now test the monitoring. Open a new browser tab and go to the UPS status page :
http://xxx.xxx.xxx.xxx/cgi-bin/nut/upsstats.cgi
Replace [xxx.xxx.xxx.xxx](http://xxx.xxx.xxx.xxx/) with the IP address of your LXC container.

Ref : https://youtu.be/vyBP7wpN72c

1. Testing
a. System shutdown
Now you can test if your system shuts down gracefully.
In Proxmox, select your PVE node → Shell.
Issue the following command :

upsmon -c fsd

Your system should shut down all its subsystems and then shutdown itself.

b. Power outage
Now you are ready to pull the AC power plug going from your AC wall outlet to the UPS.
At the moment the UPS reaches the values set in battery.runtime.low or battery.charge.low, the shutdown should be initiated.

c. Disable Beep Warning
You may not like the BEEP warning sound and want to disable it.

Check beep status :
upsc apc ups.beeper.status
enabled

Check the commands to disable beeper :
upscmd -l apc
Instant commands supported on UPS [apc]:

beeper.disable - Disable the UPS beeper
beeper.enable - Enable the UPS beeper
beeper.mute - Temporarily mute the UPS beeper
beeper.off - Obsolete (use beeper.disable or beeper.mute)
beeper.on - Obsolete (use beeper.enable)
load.off - Turn off the load immediately
load.off.delay - Turn off the load with a delay (seconds)
load.on - Turn on the load immediately
load.on.delay - Turn on the load with a delay (seconds)
shutdown.return - Turn off the load and return when power is back
shutdown.stayoff - Turn off the load and remain off
shutdown.stop - Stop a shutdown in progress
test.battery.start.deep - Start a deep battery test
test.battery.start.quick - Start a quick battery test
test.battery.stop - Stop the battery test

Disable the beeper :
upscmd apc beeper.disable
Username (root): upsadmin
Password:
OK

Check beep status :
upsc apc ups.beeper.status
disabled
