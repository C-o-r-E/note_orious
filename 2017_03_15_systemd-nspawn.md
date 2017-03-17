# March 15 2017
## Beware the ides

I wanted to run an untrusted multimedia application on linux without having to do full machine virtualization. Not being familiar with container implementations like Docker and LXC, I had some reading to do.

Some reasearch showed that SystemD has its own implementation of cgroup management called machinectl using a tool called systemd-nspawn. It is actually intended to be an abstraction allowing for management of fully virtualized machines or just containerized ones. It seems to full of really nice functionality such as being able to create an isolated snapshot of the current machine.

I ofcourse was interested in getting sound a video acceleration running with it and some googling had suggested that others have succeded with that goal.

### Basic container up and running

Most of the very basics are covered on the [archwiki](https://wiki.archlinux.org/index.php/Systemd-nspawn) but here is a quick rundown:

1. Create a directory for the machine to live in. I created `rim` in a directory called `containers`
2. Set up the contents of the container. For an arch based system: `pacstrap -c -d ~/MyContainer base --ignore linux`
3. Boot the container using `systemd-nspawn -b -D`

### Bind Mounts

For Multimedia to work we need to share some directories from the host to the container.

1. X11
  * `/tmp/.X11-unix`
2. ALSA
  * `/dev/snd`

This can be done on the command line as follows:

`systemd-nspawn -b -D rim/ --bind=/tmp/.X11-unix:/tmp/.X11-unix --bind=/dev/snd`

### cgroup device permissions

At this point even though some device files are visible, the cgroup doesnt have permission from the kernel to use them. For example running `aplay -l` will not detect any sound cards. Running `strace -e open aplay -l` will show that it fails to open() a /dev/snd/controlCx file with EPERM.

To fix this, we need to explicitly give the cgroup permissions for the device. To get sound working, I ran ` echo 'c 116:* rwm' > /sys/fs/cgroup/devices/machine.slice/machine-rim.scope/devices.allow`