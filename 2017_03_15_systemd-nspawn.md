# March 15 2017
## systemd-nspawn, ALSA and graphics

I wanted to run an untrusted multimedia application on linux without having to do full machine virtualization. Not being familiar with container implementations like Docker and LXC, I had some reading to do.

Some reasearch showed that SystemD has its own implementation of cgroup management called machinectl using a tool called systemd-nspawn. It is actually intended to be an abstraction allowing for management of fully virtualized machines or just containerized ones. It seems to full of really nice functionality such as being able to create an isolated snapshot of the current machine.

I of course was interested in getting sound and video acceleration running with it and some googling had suggested that others have succeded with that goal.

### Basic container up and running

Most of the very basics are covered on the [archwiki](https://wiki.archlinux.org/index.php/Systemd-nspawn) but here is a quick rundown:

1. Create a directory for the machine to live in. I created `rim` in a directory called `containers`
2. Set up the contents of the container. For an arch based system: `pacstrap -c -d rim --ignore linux`
3. Boot the container using `systemd-nspawn -b -D rim`

### Bind Mounts

For Multimedia to work we need to share some directories from the host to the container.

1. X11
    * `/tmp/.X11-unix`
2. ALSA
    * `/dev/snd`

This can be done on the command line as follows:

`systemd-nspawn -b -D rim/ --bind=/tmp/.X11-unix:/tmp/.X11-unix --bind=/dev/snd`

### cgroup device permissions

At this point even though some device files are visible, the cgroup doesnt have permission from the kernel to use them. For example running `aplay -l` will not detect any sound cards. Running `strace -e open aplay -l` will show that it fails to open() a /dev/snd/controlCx file with EPERM. The same command should show the same open() succeeded on the host machine followed by some ioctl()s.

To fix this, we need to explicitly give the cgroup permissions for the device. To get sound working, I ran `echo 'c 116:* rwm' > /sys/fs/cgroup/devices/machine.slice/machine-rim.scope/devices.allow`

Some explaining is required. From what I understand, `/sys/fs/cgroup/devices/` contains the kernel interface to cgroups. Every container that you run (at least with systemd-nspawn) will have its own directory inside. In my example that directory is `/sys/fs/cgroup/devices/machine.slice/machine-rim.scope/`. SystemD organizes things in to "Slices" and usually containers should show up under the `machine.slice` slice.

So inside the `/sys/fs/cgroup/devices/machine.slice/machine-rim.scope/` directory, there are two files of particular interest here. `devices.allow` and `devices.list`. You can check what permissions the cgroup already has by checking the contents of the latter: `cat /sys/fs/cgroup/devices/machine.slice/machine-rim.scope/devices.list`. The former is a *write only* file where you can modify the permissions on a per device basis. The echo command from above is an example of that.

The format for strings input in to the `devices.allow` file is: `T M:N O` where

 * T is `c` for character device, `b` for block device, etc.
 * M is major number
 * N is minor number or `*`
 * O is some combination of `r` `w` and `m` for the operations allowed

Great! With all of this, I can give my container permission to use sound and graphics! However its a pain in the ass to do all these steps each time. I could write a script to do exactly what I just described, but a better and more robust solution would be to write a systemd unit file to launch the contain as a service.

### multimedia container service

At the time of writing this, systemd has a way of running machines with `machinectl`. You just need to store your machine in one of the predefined locations that systemd looks at. It will look for machines in `/var/lib/machines/`, `/usr/local/lib/machines/` or `/usr/lib/machines/`. However it seems to not follow symbolic links. I want to store my containers in a different location so I have to do something else.

Normally you can start a machine by running `machinectl start rim` and you will be up and running. Since there are some customizations to do, a custom unit file will be needed for a custom service. I copied the provided template service `/usr/lib/systemd/system/systemd-nspawn@.service` and customized it. The naming convention is super important here. A template service will have a name like X@.service. If you start instances of the service with different names, they will show up as X@Y.service and X@Z.service. There are a few techniques for overriding specific instances of template units but I decided to entirely define one based on the above mentioned file. I saved mine as `/etc/systemd/system/systemd-nspawn@rim.service`. Here is [the service file](files/systemd-nspawn@rim.service) for reference.

Now to start the container I just do `systemctl start systemd-nspawn@rim.service`.

### DeviceAllow

Looking at [the unit file](files/systemd-nspawn@rim.service) you will see that the familiar systemd-nspawn binary as part of the ExecStart line. This time there is no `--bind` directives as they will be taken care of in the next section.  Note that the `--keep-unit` switch has to do with the cgroup appearing in the proper slice in the `/sys/fs/` hierarchy.

One of the key parts of the unit file is the DeviceAllow settings. These allow us to automate setting the permissions for cgroup device accesss without using the unwieldy `/sys/fs` inteface discussed above. A combination of this, bind mounts and X11 forwarding allows for the goal to be met! 

### .nspawn config files

When starting a container, systemd-nspawn will look for an additional settings file in `/etc/systemd/nspawn`. It will attempt reading a file that corresponds to the name of the machine, so in my case it will be `/etc/systemd/nspawn/rim.nspawn`.

Here is my [.nspawn config](files/rim.nspawn).

It is very simple. There are 3 groups of bindings.

 * X11
 * ALSA
 * Graphics

With this file we are able to be a little cleaner than jamming all the directory binding in to the invokation of `systemd-nspawn`. Instead each bind now has its own line in the .nspawn file. For example `Bind=/dev/snd`.

Note that my config file is set up for an Nvidia card. It will be different for ATI and Intel graphics devices.


### the final bits

Almost there! Applications in this container should now be able to talk to the sound hardware and the graphics card. We need to be able to talk to X11 though! To do this we need to do 2 things:

1. allow X11 on the host to listen for local clients
  * Accomplished by running  `xhost +local:` on the host
2. tell the container's apps to use the correct display
  * either run `export DISPLAY=:0` from the container shell
  * or add the export command to a startup script such as `.bashrc`

### fin

I think that's all. Let's call this a first draft?





## Useful Links:

[https://www.kernel.org/doc/Documentation/cgroup-v1/devices.txt](https://www.kernel.org/doc/Documentation/cgroup-v1/devices.txt)

[http://www.makelinux.net/ldd3/chp-3-sect-2](http://www.makelinux.net/ldd3/chp-3-sect-2)

[https://www.freedesktop.org/wiki/Software/systemd/ControlGroupInterface/](https://www.freedesktop.org/wiki/Software/systemd/ControlGroupInterface/)

[https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#DeviceAllow=](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#DeviceAllow=)

[https://www.freedesktop.org/software/systemd/man/systemd.nspawn.html#](https://www.freedesktop.org/software/systemd/man/systemd.nspawn.html#)

[https://www.freedesktop.org/software/systemd/man/machinectl.html#](https://www.freedesktop.org/software/systemd/man/machinectl.html#)

[https://wiki.archlinux.org/index.php/Change_root#Run_graphical_applications_from_chroot](https://wiki.archlinux.org/index.php/Change_root#Run_graphical_applications_from_chroot)

[https://wiki.archlinux.org/index.php/Systemd-nspawn](https://wiki.archlinux.org/index.php/Systemd-nspawn)

[Stack Overflow question](http://unix.stackexchange.com/questions/304252/access-usb-device-from-systemd-nspawn-container)

[https://wiki.archlinux.org/index.php/cgroups](https://wiki.archlinux.org/index.php/cgroups)
