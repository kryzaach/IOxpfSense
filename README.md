# IOxpfSense
Running a firewall in a switch - KVM with L2 interfaces on Cat9k

This was a painful excercise so I'm taking the time to note what needed to be done to get this going.

# Required: 
  - A machine with a different kernel that you want to virtualise on a Cisco Catalyst 9000 series switch
    
    i) For me, this is pfSense (FreeBSD), on a C9300-48UXM-A
  - The correct licenses for whatever you're trying to do
    
    i) DNA-Advantage is required to use IOx
  - A Type 2 Hypervisor to build your "Application"
    
    i) I'm using Vmware Workstation
    
# Things you need to know:
  - How to build a VM of your chosen machine, I'm not going to go into any special detail here apart from preparing it for use in the IoX environment
  - What IOx is and is not, a good introduction is here: https://developer.cisco.com/docs/iox/#!introduction-to-iox
  - Familiarity with IoS XE and networking in general
  - Familiarity with Hypervisors

First and foremost, there is almost no documentation on this process, so be warned. I read several slide decks from Cisco Live that vaguely alluded to capabilities and features that kept me focused through 10 IoS upgrades and downgrades, 5 VM rebuilds and at least 100 package rebuilds.

# Broad steps:
  - Build VM in hypervisor
  - Add necessary packages to VM (qemu)
  - Set up VM with correct network configuration
  - Enable serial console for VM
  - Convert VM to qcow2
  - Create IOx package
  - Create application configuration on switch
  - Deploy IOx package, activate and start
  - ????
  - Profit

# Detailed process (pfSense example):

Download the Community edition ISO installer (AMD64) from here https://www.pfsense.org/download/ and create a VM in your chosen hypervisor.

Customise the hardware prior to boot, by adding a second network card and a serial port. For Vmware Workstation on Windows, redirect this to a named pipe and connect putty to the same named pipe after starting the machine.

Follow the guided installed and set up 2 network interfaces without VLANs(unless you want to, but that is left as an excercise to the reader), and designate one WAN and one LAN. Set the WAN interface to be addressed via DHCP and bridge it to a network with internet access (typically your host's main adapter). Set the designated LAN interface address to a free address in your "Host Only" subnet for your Hypervisor and complete the install.

Jump to a Shell (Option 8) when that is complete.

The next few steps mainly come from here: https://forum.netgate.com/topic/162083/pfsense-vm-on-proxmox-qemu-agent-installation

Run the following command:

`pkg install qemu-guest-agent`

Edit /etc/rc.conf.local and add the following:

```
qemu_guest_agent_enable="YES"

qemu_guest_agent_flags="-d -v -l /var/log/qemu-ga.log"
```

Browse to the web gui via the LAN IP you set previously and install "Shellcmd" from "System/PackageManager"

Under "Service/Shellcmd", create an "earlyshellcmd" as follows:

`service qemu-guest-agent start`

Under "System/Settings/Advanced/Tunables", add the following:

`Tunable: virtio_console_load, Value: YES`

Under "System/Advanced", tick "Enable the first serial port with 115200/8/N/1".

Create a linux VM, I was using Kali, so instructions will work for most debian based distros.

Locate the firewall VM files and copy the .vmdk file across to your linux environment.

The following steps mainly come from here: https://developer.cisco.com/docs/iox/#!tutorial-build-sample-vm-type-iox-app/tutorial-build-sample-vm-type-iox-app

Install the qemu utils:

`apt install qemu-utils`

Convert the disk to qcow2 format:

`qemu-img convert -c -O qcow2 <mymachine>.vmdk <mymachine>.qcow2`

Create a package.yaml file that specifies the machine config:

```
descriptor-schema-version: "2.4"

info:
  name: pfSense 
  description: "Virtual firewall on IOx"
  version: "1.0"
  author-link: "http://www.cisco.com"
  author-name: "Cisco Systems"

app:
  type: vm
  cpuarch: x86_64
  resources:
    profile: custom
    cpu: 7400 
    memory: 2048
    vcpu: 2
    disk: 20000

    network:
      -
        interface-name: eth0
      -
        interface-name: eth1

  # Specify runtime and startup
  startup:
    disks:
      -
        target-dev: "hdc"
        file: "<mymachine>.qcow2"

    qemu-guest-agent: True
```
This file format is not described well anywhere as far as I can determine is a fork of: https://github.com/naoki2001/kvm-compose

This could just be my ignorance though, however the following things are important:

Even though it is not used, the disk must be the same size as your VMDK image in MB otherwise you will hit an error later: `Error while changing app state: ('Failed to allocate dd for %s`

The maximum available CPU units on a C9300 platform is 7400, exceeding this will give you a CPU capacity error and reject the package deployment

Download the latest ioxclient package for your build environment.

Copy the package.yaml file you created above, as well as the qcow2 image of your VM into a new folder. I used `output-dir`.

Run the following command:

`ioxclient profiles create`

Then run:

`ioxclient package --name <mymachine> output-dir`

This will create a file <mymachine.tar> in the output directory.

Copy that tar file back out to a machine that has access to your switch.

IOx is meant to be API driven and has a bunch of fancy capabilities in that regard, but we're going to ignore them and use the CLI.

***THIS IS THE MOST IMPORTANT BIT AND WASTED HOURS OF MY LIFE***

Down/upgrade your switch to Gibraltar 16.12.01

There are probably others that you can use, but I tried a variety and I can tell you the following:

Fuji 16.9.1 is too early as the AppGigabitEthernet interface doesn't show up (but I have seen it in the r1 release)

Gibraltar 16.12.07 is too late as the ability to use real L2 bridging with VMs has been removed.

If you're later than that you'll get one of a variety of errors al;ong the lines of: `Error: Cannot find device "L2br"`

Enable IOx:

```
conf t
iox
```

Copy the package over to the switch using TFTP or plug in a USB

Deploy the package:

`app-hosting install appid <mymachine>package usbflash0:<mymachine>.tar`

*IF* you've done everything right you'll get a message as follows:

```
<mymachine> installed successfully
Current state is: DEPLOYED
```

As you're building a firewall, we'll assume you have an inside and outside vlan. I this example, these are 3 and 2 respectively.

Use the following configuration to ensure that communication will work as expected:

```
interface AppGigabitEthernet1/0/1
 switchport trunk allowed vlan 2,3
 switchport mode trunk
end
```

Use the following configuration to make sure the machine is described to IOx correctly:

```
alias exec pfsense app-hosting connect appid <mymachine> console
app-hosting appid <mymachine>
 app-vnic AppGigabitEthernet trunk
  vlan 2 guest-interface 0
  vlan 3 guest-interface 1
 app-resource profile custom
  cpu 7400
  memory 2048
  vcpu 2
```

Move the machine to the activated state:

`app-hosting activate appid <mymachine>`

You should get the following message:

```
<mymachine> activated successfully
Current state is: ACTIVATED
```

Start the machine:

`app-hosting start appid <mymachine>`

You should get the following message:

```
<mymachine> started successfully
Current state is: RUNNING
```

Now just call the alias you created previously:

`pfsense`

You should get a nice friendly pfSense console:

```
Connected to appliance. Exit using ^c^c^c



FreeBSD/amd64 (<machine fqdn>) (ttyu0)

KVM Guest - Netgate Device ID: <id>

*** Welcome to pfSense 2.6.0-RELEASE (amd64) on <mymachine> ***

 WAN (wan)       -> pppoe1     -> v4/PPPoE: ***.***.***.***/32
 LAN (lan)       -> vtnet1     -> v4: 192.168.1.1/24

 0) Logout (SSH only)                  9) pfTop
 1) Assign Interfaces                 10) Filter Logs
 2) Set interface(s) IP address       11) Restart webConfigurator
 3) Reset webConfigurator password    12) PHP shell + pfSense tools
 4) Reset to factory defaults         13) Update from console
 5) Reboot system                     14) Enable Secure Shell (sshd)
 6) Halt system                       15) Restore recent configuration
 7) Ping host                         16) Restart PHP-FPM
 8) Shell

Enter an option:
```

You may need to use option 2 to readdress things, but you can now drive your pfSense isntance as per usual.


