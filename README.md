# archlinux-virtualbox-host


## Content

- [About](#about)
- [Motivation](#motivation)
- [How to start](#how-to-start)
  - [At the Docker host](#At-the-Docker-host)
  - [Create a new guest machine](#create-a-new-guest-machine)
  - [Accessing the guest OS with an RDP viewer](#accessing-the-guest-os)
- [Next steps](#Next-steps)
- [Description](#Description)
  - [Dockerfile](#Dockerfile)
    - [The *entrypoint* script](#entrypoint)
  - [Environment variables](#variables)
- [Notes](#Notes)
  - [Why Arch Linux image](#Why-Arch-Linux)
  - [The VirtualBox terms of use](#VirtualBox-terms-of-use)

  
<a name="about"></a>

## About

This repo provides Docker compose and image files definitions
to pull or build an image and start a docker container running Arch Linux
with [*Oracle VM VirtualBox*](https://www.virtualbox.org/) host.

If an existing VirtualBox guest machine,
represented by a folder with a `*.vbox` file and its dependencies,
is mounted into the container, the container can automatically start
and gracefully stop the guest machine.

<a name="the-guest-machine"></a>
The *guest machine* folder with the machine configuration file `*.vbox`
can be created elsewhere with
[*VirtualBox GUI*](https://www.virtualbox.org/manual/ch01.html#gui-createvm)
or with [*vboxmanage createvm*](#interactive-bash-session)
in an interactive [*bash*](#interactive-bash-session) session in the container.

The running guest machine can be [accessed](#accessing-the-guest-os)
with an RDP client.


<a name="motivation"></a>

## Motivation

- Avoid the installation of a complete `virtualbox` package in a *headless* Arch Linux host.\
Along with the *cli* VirtualBox interface, the package includes *Qt GUI*, irrelevant in a headless host.

- Automate the startup/shutdown of a VirtualBox guest OS by Docker daemon
(and thereby control the availability of the services provided by the guest OS)


<a name="how-to-start"></a>

## How to start


<a name="At-the-Docker-host"></a>

**At the [Docker host](https://docs.docker.com/get-started/overview/#docker-architecture)**
(*running under Arch Linux*, it can be remote and headless)

- Install the VirtualBox drivers

   ```shell
   pacman -S virtualbox-host-modules-arch
   ```

- Override the `docker.service` configuration as [suggested](#docker-service-overriding).

- Clone this repo

   ```shell
   git clone "https://github.com/o124/archlinux-virtualbox-host.git"
   cd "archlinux-virtualbox-host"
   ```

- Modify values in the [`.env`]() file if needed.
  Note, modification of IMAGE_NAME, VMDIR, VBUSR, VBUID, and TZONE will require rebuilding of the docker image.
  Generally, the default values should be acceptable, at least for testing.
  With the default values a prebuilt image can be pulled from the dockerhub
  [repo](https://hub.docker.com/r/o124/archlinux-virtualbox-host).

- Pull/build the image

   ```shell
   sudo docker compose build
   ```

- If you have your [guest machine](#the-guest-machine) already prepared,
  copy it into [`$VMSRC`](#VMSRC).
  Along with the `*.vbox` file, all the `*.iso`, `*.vdi`, and other files it refers to
  should also be in `$VMSRC`.
  If you do not have the guest machine yet,
  [create a new one](#create-a-new-guest-machine), then continue from here.
  Make sure that the `*.vbox` configuration [allows](#accessing-the-guest-os)
  incoming RDP connections.

- <a name="make-VMSRC-accessible"></a>
  Provide a read/write access for [`$VBUID`](#VBUID) to [`$VMSRC`](#VMSRC).
  The included helper script [`set-perm-vmsrc`]() can be used at the Docker host:

   ```shell
   sudo ./set-perm-vmsrc
   ```

- Start the service

   ```shell
   sudo docker compose up -d
   sudo docker compose logs -f
   ```

  The container logs should show something like this

   ```
   vbox-base  | Info     Found VM "arch"
   vbox-base  | Info     Launching "arch"
   vbox-base  | Info     VBoxHeadless subshell PID=62
   vbox-base  | Starting virtual machine: 10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
   vbox-base  | Info     "arch" state is "running"
   vbox-base  | Info     "arch" has started
   vbox-base  | Info     Suspend entrypoint:run() while VBoxHeadless subshell PID=62 is running
   ```

Now the guest machine should be [accessible](#accessing-the-guest-os)
for RDP clients.

Then, when the guest OS shuts down properly,
for example, when a user shuts it down interactively,
the following appears in the container logs

   ```
   vbox-base  | Oracle VM VirtualBox Headless Interface 6.1.36
   vbox-base  | (C) 2008-2022 Oracle Corporation
   vbox-base  | All rights reserved.
   vbox-base  |
   vbox-base  | VRDE server is listening on port 3389.
   vbox-base  | VRDE server is inactive.
   vbox-base  | Info     VBoxHeadless subshell PID=62 exited, EC=0
   vbox-base  | Info     Resuming entrypoint:run()
   vbox-base  | Info     Waiting for VBoxHeadless to exit
   vbox-base  | Info     Waiting for the background VBox processes to terminate...
   vbox-base  | Info     All done
   vbox-base  |
   vbox-base exited with code 0
   ```

A normal guest OS shutdown can also be triggered by Docker host commands:

   ```shell
   sudo docker compose stop
   ```

Then the container logs show

   ```
   vbox-base  | Info     Signal TERM trapped
   vbox-base  | Info     "arch" state is "running"
   vbox-base  | Info     Sending ACPI power button signal to "arch"
   vbox-base  | Info     "arch" received 'acpipowerbutton' signal
   vbox-base  | Info     Trap TERM finished
   vbox-base  | Info     VBoxHeadless subshell PID=62 exited, EC=143
   vbox-base  | Info     Resuming entrypoint:run()
   vbox-base  | Info     Waiting for VBoxHeadless to exit
   vbox-base  | Oracle VM VirtualBox Headless Interface 6.1.36
   vbox-base  | (C) 2008-2022 Oracle Corporation
   vbox-base  | All rights reserved.
   vbox-base  |
   vbox-base  | VRDE server is listening on port 3389.
   vbox-base  | VRDE server is inactive.
   vbox-base  | Info     Waiting for the background VBox processes to terminate...
   vbox-base  | Info     All done
   vbox-base  |
   vbox-base exited with code 0
   ```

Clearly, the same happens when the `docker.service` stops,
for instance because the Docker host computer shuts down or reboots.


<a name="create-a-new-guest-machine"></a>

### Create a new guest machine from withing the container.

Make sure, [`$VMSRC`](#VMSRC) at the Docker host is
[accessible](#make-VMSRC-accessible)
to [`$VBUID`](#VBUID).

<a name="interactive-bash-session"></a>

*To start an interactive bash session* in the container instead of the guest OS,
run

```shell
sudo docker compose run -it --rm --name vbox vbox bash
```

A container should start up and get ready for input.

*In the container*, use [`vboxmanage`](https://www.virtualbox.org/manual/ch08.html#vboxmanage-createvm)
to create a new guest machine and modify it appropriately.

Use `$VMDIR` for `--basefolder` parameter with the `createvm` command.

An example script [`vm_create`]() to create Arch Linux guest is included here.
To use it, modify it following comments in the script file and run it in the container

```shell
cp /usr/local/bin/vm_create vm_create
# edit the file at the Docker host, 
# it is now in the respective named volume location 
bash vm_create
```

When the machine is created and modified as intended, exit from the bash session.


<a name="accessing-the-guest-os"></a>

### Accessing the guest OS with an RDP viewer

**At the Docker host**
<span style="font-size: 0.95em">
(before starting the guest machine)
</span>.

- To activate the VirtualBox *VRDP* server and allow incoming *RDP* connections
  the guest machine configuration file `*.vbox`
  should have in the `<Hardware/>` section lines like these

   ```xml
      <RemoteDisplay enabled="true">
        <VRDEProperties>
          <Property name="TCP/Ports" value="3389"/>
        </VRDEProperties>
      </RemoteDisplay>
   ```

  With the default *VRDP* port number of 3389, the `<VRDEProperties/>` block should be redundant.
  The configuration can be adjusted with
  [*VirtualBox GUI*](https://www.virtualbox.org/manual/ch03.html#settings-remote-display)
  or with
  [`vboxmanage`](https://www.virtualbox.org/manual/ch08.html#vboxmanage-modifyvm-vrde)
  command.

- The [`compose.yml`]()
  file should map the [`$VRDP`](#VRDP) port from Docker container to the Docker host
  <span style="font-size: 0.95em">
  (the provided configuration does it)
  </span>.


**At the RDP client machine** (*"headed"*)

- install an RDP client, for instance [*freerdp*](https://www.freerdp.com/).

   ```shell
   pacman -S freerdp
   ```

- if the RDP client machine is the same as the Docker host machine,\
  connect the RDP client to the running guest OS *'directly'*.

   ```shell
   # substitute $VRDPIP and $VRDP with the values from .env file
   xfreerdp +v:$VRDPIP:$VRDP +video +clipboard
   ```

- if the RDP client machine is not the same as the Docker host machine,

   - forward the `$VRDP` port from the Docker host machine:

      ```shell
      ssh -NL <Client-Local-IP>:<Client-Local-VRDP>:<$VRDPIP>:<$VRDP> <Docker-host-machine>
      # for example:
      # ssh -NL 127.0.0.1:3389:127.0.0.1:3389 192.168.0.1
      ```

   - connect the RDP client to the running guest OS:

      ```shell
      # substitute <Client-Local-IP> and <Client-Local-VRDP>
      # for the values from the ssh command above
      xfreerdp +v:<Client-Local-IP>:<Client-Local-VRDP> +video +clipboard
      ```


<a name="Next-steps"></a>

## Next steps

- After initial tests confirm expected container behaiviuor,
  the `restart:` configuration entry in `compose.yaml`
  can be changed to `unless-stopped`:

  ```yaml
  restart: unless-stopped
  ```

- If your guest OS exposes network services,
  and the `*.vbox` configuration exposes them via VirtualBox NAT system,
  for instance, like this

   ```xml
   <NAT>
      <Forwarding name="MSSQLS" proto="1" hostport="1433" guestport="1433"/>
   </NAT>
   ```
  to make them available at the Docker host system
  add corresponting `ports:` entries in `compose.yml`

   ```yaml
   ports:
     - 127.0.0.1:1433:1433
   ```


<a name="Description"></a>

## Description

<a name="Dockerfile"></a>

### The Dockerfile

<!-- The image is based on [`archlinux`](https://hub.docker.com/_/archlinux). -->
The image is based on [`archlinux/archlinux`](https://hub.docker.com/r/archlinux/archlinux).
[Here](#Why-Arch-Linux) is why.


It adds the packages

- `virtualbox`
- `virtualbox-host-modules-arch`
- `virtualbox-guest-iso`

and the local scripts

- `/usr/local/bin/setup`
- `/usr/local/bin/entrypoint`

The [environment](#variables) variables and arguments described below
define the generated image.

The `setup` script is used in the Dockerfile to

- download and install the *Oracle VM VirtualBox Extension Pack*
- create user [`$VBUSR`](#VBUSR) with [`$VBUID`](#VBUID)
- create [`$VMDIR`](#VMDIR) and set its permissions
- set the timezone to [`$TZONE`](#TZONE)

The `entrypoint` script is described [below](#entrypoint).


The Dockerfile exposes volumes

- [`$VMDIR`](#VMDIR), a mount point for the guest machine folder
- [`/home/$VBUSR`](#VBUSR) that holds the guest machine registration and other VirtualBox data

and ports

- [`$VRDP`](#VRDP), an *RDP* port used to access the running guest machine.


To run VirtualBox commands with lesser privileges,
the Dockerfile executes the `HEALTHCHECK` and `ENTRYPOINT` instructions
as user [`$VBUSR`](#VBUSR).

The default `HEALTHCHECK` here only checks
[*MachineState*](https://www.virtualbox.org/sdkref/_virtual_box_8idl.html#a80b08f71210afe16038e904a656ed9eb)
reported by `vboxmanage showvminfo`.\
If the `arch/dist/bin/healthcheck` file (not provided here) existed during the image generation,\
it is executed instead and its exit code is reported to the `HEALTHCHECK` instruction.\
This custom `healthcheck` can do more guest-relevant tests.
For example, it can check availability of the services provided by the guest.


<a name="entrypoint"></a>

#### The *entrypoint* script.

The `entrypoint` script controls the guest machine registration and life cycle.
It is the main part of the repo.

On every start, the `entrypoint` script gets the list of the registered guests
from `vboxmanage list vms`, and uses the first one
(the only one is expected to be there) to start it with `vboxheadless --startvm`.

<a name="if-no-registered-machine-found"></a>
If no registered machine found (for instance, on the first start),
the script recursively scans [`$VMDIR`](#VMDIR) to find all `*.vbox` files,
sorts them alphabetically, then
uses the first one to register it with `vboxmanage registervm`
(therefore, just one registered guest is supposed to be there),
and try again with `vboxmanage list vms` and `vboxheadless --startvm`.

After the guest is successfully started,
the script waits for the `VBoxHeadless` process to exit
(this happens for instance when the guest OS stops),
then it waits for the remaining `VBoxSVC` and `VBoxXPCOMIPCD` processes to exit,
then the container stops running.

---

A safe guest OS shutdown can also be triggered by Docker host system events.

When [docker daemon](https://docs.docker.com/get-started/overview/#docker-architecture)
tries to stop the container, it sends the `TERM` signal to the container,
waits for [`$GRACE_PERIOD`](#GRACE_PERIOD) until it stops gracefully,
and if the container does not stop afterwards, kills it.

To allow a graceful shutdown of the guest OS withing the `$GRACE_PERIOD`,
the `entrypoint` script traps the `INT`, `QUIT`, and `TERM` signals
and in response sends `acpipowerbutton` signal to the guest OS.
*If the guest OS handles the ACPI signal*,
it should be able to shut down safely and let the `VBoxHeadless` process to
exit and let the container stop normally.

<a name="docker-service-overriding"></a>
Note, `docker.service` at the docker host requires
[overriding](https://wiki.archlinux.org/title/systemd#Drop-in_files) of the
[`TimeoutStopSec=`](https://www.freedesktop.org/software/systemd/man/systemd.service.html#TimeoutStopSec=)
value.

   ```unit file (systemd)
   # /etc/systemd/system/docker.service.d/TimeoutStopSec.conf

   [Service]
   # Set the appropriate value according to your guest OS needs
   TimeoutStopSec=5h
   ```

Otherwise, while stopping the `docker.service`,
`systemd` will kill docker and guest OS after the default 90 seconds
which could be not sufficient for some guest OS to stop properly.

---

The `entrypoint` script recognises the following _commands_
(provided as the only script parameter):

- `run`, start the guest machine as described [above](#entrypoint).
   This is *the default* action if no any command is specified.
- `list`, show *name* and *id* of the registered guest machine reported by `vboxmanage list vms`,
   *do not* start the guest machine.
- `state`, show *VMState* from `vboxmanage showvminfo`,
   *do not* start the guest machine.
- `healthcheck`, implement a simple default `HEALTHCHECK` for Dockerfile:\
   return success (meaning healthy container state) if *VMState* is one of \
   *running*|*restoring*|*saving*|*paused*|*saved*.
   *Do not* start the guest machine.

The _commands_ can be used in the
[`command:`](https://docs.docker.com/compose/compose-file/#command)
configuration entry in `compose.yaml`

Unrecognised commands are executed with `exec "$@"`
replacing thereby the `entrypoint` process.
This can be used to execute arbitrary commands in the container,
for example see [here](#interactive-bash-session).


<a name="variables"></a>

### Environment variables

The default values for the environment variables are defined in the
[`.env`]() file.

<a name="IMAGE_NAME"></a>
**$IMAGE_NAME** sets the image name
[[ref]](https://docs.docker.com/compose/compose-file/#image).

<a name="VBGUEST"></a>
**$VBGUEST** is a label used to construct the container name,
and `$COMPOSE_PROJECT_NAME`.\
See how in [`.env`]() and [`compose.yaml`]()

<a name="COMPOSE_PROJECT_NAME"></a>
**$COMPOSE_PROJECT_NAME**
[[ref]](https://docs.docker.com/compose/reference/envvars/#compose_project_name)
is included in the folder names for the named volumes.

<a name="VMSRC"></a>
**$VMSRC** is mounted into [`$VMDIR`](#VMDIR) inside the container when it starts.
It should contain a complete VirtualBox machine to run, i.e.
a `*.vbox` file defining the machine and everything it refers to.
`$VMSRC` can also be a parent of the machine folder.
`$VMSRC` should have permissions allowing `$VBUSR` with `$VBUID` inside the container
to read and modify it.
The included helper script [`set-perm-vmsrc`](), making use of the file `.env`,
can be executed in the Docker host system
to set the appropriate permissions on `$VMSRC` before starting the container.

<a name="VMDIR"></a>
**$VMDIR** is a directory inside the container
where [`$VMSRC`](#VMSRC) is to be mounted.
See also [here](#if-no-registered-machine-found).

<a name="VBUSR"></a>
**$VBUSR** is a linux username used to run the container.
It does not have to exist in the docker host system.

<a name="VBUID"></a>
**$VBUID** is a linux user ID of [`$VBUSR`](#VBUSR).
It does not have to exist in the docker host system.

<a name="TZONE"></a>
**$TZONE** is used to set `/etc/localtime` in the image like
[here](https://wiki.archlinux.org/title/Installation_guide#Time_zone).

<a name="GRACE_PERIOD"></a>
**$GRACE_PERIOD**
[[ref]](https://docs.docker.com/compose/compose-file/#stop_grace_period)
should be long enough to let your guest OS to stop properly
when the container is instructed to stop.

<a name="VRDPIP"></a>
**$VRDPIP**
is an IP address at the Docker host to bind [`VRDP port`](#VRDP) to.
It is where the service will listen for the incoming RDP client connections.

<a name="VRDP"></a>
**$VRDP**
is a port number that should be set to the same *VRDP* port number
that is set in `.vbox` file, see
[here](#accessing-the-guest-os).


<a name="Notes"></a>

## Notes

<a name="Why-Arch-Linux"></a>

### Why Arch Linux image

VirtualBox host running in Docker container uses VirtualBox drivers from Docker host,
`/dev/vboxdrv` being one of them.
So, if Docker host is running under Arch Linux,
having the VirtualBox host and the VirtualBox drivers
from the same distribution source and of the same version
should favour the system stability.


<a name="VirtualBox-terms-of-use"></a>

### The VirtualBox terms of use

About the VirtualBox and VirtualBox Extension Pack licenses see
[here](https://www.virtualbox.org/wiki/Licensing_FAQ)
