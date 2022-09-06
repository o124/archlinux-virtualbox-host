# archlinux-virtualbox-host


## Content

- [About](#about)
- [Motivation](#motivation)
- [How to start](#how-to-start)
  - [At the Docker host](#At-the-Docker-host)
  - [Create a new guest machine](#create-a-new-guest-machine)
- [Accessing the guest OS with an RDP client](#accessing-the-guest-os)
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

This repo provides docker compose and image file definitions
to pull or build an image and start a docker container running
[*Oracle VM VirtualBox*](https://www.virtualbox.org/)
host in Arch Linux.

If an existing VirtualBox *guest* machine,
represented by a folder with a `*.vbox` file and its dependencies,
is mounted into the container, the container can automatically start
and gracefully stop the guest machine.

<a name="the-guest-machine"></a>
The guest machine folder with the machine configuration file `*.vbox`
can be created with [vboxmanage createvm](https://www.virtualbox.org/manual/ch08.html#vboxmanage-createvm)
in the container ([see below](#create-a-new-guest-machine)) or elsewhere with
[VirtualBox GUI](https://www.virtualbox.org/manual/ch01.html#gui-createvm).

The running guest machine can be [accessed](#accessing-the-guest-os)
with an RDP client.

The corresponding dockerhub
[repo](https://hub.docker.com/r/o124/archlinux-virtualbox-host)
is built from the sources herein.


<a name="motivation"></a>

## Motivation

- Avoid the installation of a complete `virtualbox` package
  in a *headless* Arch Linux host.

- Automate the startup/shutdown of VirtualBox guest machines by the Docker daemon.

- Provide an alternative response to the `TERM` signal for `VBoxHeadless`.
  By default, VBoxHeadless tries to `savestate` of the guest machine,
  which is not always the needed action.


<a name="how-to-start"></a>

## How to start


<a name="At-the-Docker-host"></a>

A *[Docker host](https://docs.docker.com/get-started/overview/#docker-architecture)*
running under Arch Linux is implied here, it can be remote and headless.

**At the Docker host**:

- Install the VirtualBox drivers

   ```shell
   sudo pacman -S VIRTUALBOX-HOST-MODULES
   ```

- Override the `docker.service` configuration as
  [suggested](#docker-service-overriding).


- Clone this repo

   ```shell
   git clone "https://github.com/o124/archlinux-virtualbox-host.git"
   cd "archlinux-virtualbox-host"
   ```

- Modify values in the
  [`.env`](https://github.com/o124/archlinux-virtualbox-host/blob/main/.env)
  file if needed.
  Note, modification of IMAGE_NAME, VMDIR, VBUSR, VBUID, and TZONE
  will require rebuilding of the docker image.
  Generally, the default values should be acceptable, at least for testing.
  An image built with the default values can be pulled from the dockerhub
  [repo](https://hub.docker.com/r/o124/archlinux-virtualbox-host).


- Either pull the prebuilt image

   ```shell
   sudo docker compose pull
   ```

  or build a custom one

   ```shell
   sudo docker compose build
   ```

- If a guest machine is already prepared,
  copy it into [`$VMSRC`](#VMSRC).
  Along with the machine configuration `*.vbox` file,
  all the `*.iso`, `*.vdi`, and other files it refers to
  should also be copied into [`$VMSRC`](#VMSRC).
  Make sure that the `*.vbox` configuration [allows](#accessing-the-guest-os)
  incoming RDP connections.
  If you do not have the guest machine yet,
  [create a new one](#create-a-new-guest-machine), then continue from here.


- <a name="make-VMSRC-accessible"></a>
  Provide a read/write access to [`$VMSRC`](#VMSRC) for [`$VBUID`](#VBUID).
  For that, the included helper script
  [`set-perm-vmsrc`](https://github.com/o124/archlinux-virtualbox-host/blob/main/set-perm-vmsrc)
  making use of the
  [`.env`](https://github.com/o124/archlinux-virtualbox-host/blob/main/.env)
  file can be used at the Docker host:

   ```shell
   sudo ./set-perm-vmsrc
   ```

- Start the service defined in the
  [`compose.yml`](https://github.com/o124/archlinux-virtualbox-host/blob/main/compose.yml)
  file and follow log messages

   ```shell
   sudo docker compose up -d
   sudo docker compose logs -f
   ```

  The container log should show something like this
  (here and below, "arch" is the demo guest machine name)

   ```
   vbox-base  | Info     Launching "arch"
   vbox-base  | Info     VBoxHeadless subshell PID=61
   vbox-base  | Starting virtual machine: 10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
   vbox-base  | Info     "arch" has started
   vbox-base  | Info     Suspend entrypoint:run() while VBoxHeadless subshell PID=61 is running
   ```

Now the guest machine should be accessible for RDP clients,
[here](#accessing-the-guest-os) is how.

When the guest OS shuts down properly,
for example, when a user shuts it down using the guest OS interface,
the following appears in the container log

   ```
   vbox-base  | Oracle VM VirtualBox Headless Interface 6.1.36
   vbox-base  | (C) 2008-2022 Oracle Corporation
   vbox-base  | All rights reserved.
   vbox-base  |
   vbox-base  | VRDE server is listening on port 3389.
   vbox-base  | VRDE server is inactive.
   vbox-base  | Info     VBoxHeadless subshell exited with EC=0
   vbox-base  | Info     Resuming entrypoint:run()
   vbox-base  | Info     Waiting for VBoxHeadless to exit
   vbox-base  | Info     Waiting for the background VBox processes to terminate...
   vbox-base  | Info     All done
   vbox-base exited with code 0
   ```

Alternatively, a normal guest OS shutdown can also be triggered by Docker host commands:

   ```shell
   sudo docker compose stop
   ```

Then the container log shows

   ```
   vbox-base  | Info     Signal TERM trapped
   vbox-base  | Info     "arch" state is "running"
   vbox-base  | Info     Sending 'acpipowerbutton' signal to "arch"
   vbox-base  | Info     "arch" received 'acpipowerbutton' signal
   vbox-base  | Info     Trap TERM finished
   vbox-base  | Info     VBoxHeadless subshell exited with EC=143
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
   vbox-base exited with code 0
   ```

The same happens when the `docker.service` stops,
for instance because the Docker host computer shuts down or reboots.


<a name="create-a-new-guest-machine"></a>

### Create a new guest machine using the container

[Make sure](#make-VMSRC-accessible) that [`$VMSRC`](#VMSRC) at the Docker host
is accessible to [`$VBUID`](#VBUID).

<a name="interactive-bash-session"></a>

To start an *interactive bash session* in the container instead of the guest machine,
temporarily disable
[`healthcheck`](https://docs.docker.com/compose/compose-file/#healthcheck)
in the [`compose.yml`](https://github.com/o124/archlinux-virtualbox-host/blob/main/compose.yml)
file

```yaml
  healthcheck:
    disable: true
```

and run

```shell
sudo docker compose run -it --rm --name base vbox bash
```

A container should start up and show bash prompt ready for input.

- Use
  [`vboxmanage createvm`](https://www.virtualbox.org/manual/ch08.html#vboxmanage-createvm)
  and
  [`vboxmanage modifyvm`](https://www.virtualbox.org/manual/ch08.html#vboxmanage-modifyvm)
  there to create a new guest machine and modify it appropriately.
  Use `$VMDIR` for `--basefolder` parameter with the `createvm` command.


- Alternatively, an example script
  [`vm-create`](https://github.com/o124/archlinux-virtualbox-host/blob/main/arch/dist/bin/vm-create)
  to create Arch Linux guest is included here.\
  To use it as it is simply run it in the container

  ```shell
  ./vm-create
  ```

  The script supports a number of variables (described inside the script) to control
  the guest machine configuration. For example, to create an ubuntu guest run:

  ```shell
  VM_NAME="s-proxy" \
  VM_OSTYPE="Ubuntu_64" \
  OS_ISO_URL="https://releases.ubuntu.com/22.04.1/ubuntu-22.04.1-live-server-amd64.iso" \
  ./vm-create
  ```

  The script can also be modified directly at the Docker host machine
  and then started in the container as above.
  To locate the named volume folder with the file at the Docker host machine do:

  ```shell
  sudo docker volume ls --format "{{.Mountpoint}}" | grep -sP 'vbox.+_vbusr'
  ```

  Here is the default result:

  ```
  /var/lib/docker/volumes/vbox-base_vbusr/_data
  ```

  There, the `./vm-create` file can be modified with any text editor available at the Docker host.

When the machine is created and modified as intended, exit from the bash session.

Finally, in the [`compose.yml`](https://github.com/o124/archlinux-virtualbox-host/blob/main/compose.yml)
file, re-enable the
[`healthcheck`](https://docs.docker.com/compose/compose-file/#healthcheck)

```yaml
  healthcheck:
    disable: false
```


<a name="accessing-the-guest-os"></a>

## Accessing the guest OS with an RDP client

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
  command (also from within the container, see [here](#create-a-new-guest-machine)).


- The [`compose.yml`](https://github.com/o124/archlinux-virtualbox-host/blob/main/compose.yml)
  file should map the [`$VRDP`](#VRDP) port from Docker container to the Docker host
  <span style="font-size: 0.95em">
  (the provided configuration does it)
  </span>.


**At the RDP client machine** (*"headed"*)

- Install an RDP client, for instance [*freerdp*](https://www.freerdp.com/).

   ```shell
   sudo pacman -S freerdp
   ```

- If the RDP client machine is the same as the Docker host machine,
  connect the RDP client to the running guest OS *'directly'*:

   ```shell
   xfreerdp +v:127.0.0.1:3389 +video +clipboard
   ```

  Here, the `+v:` parameter takes a `$VRDPIP:$VRDP` pair,
  the default values `127.0.0.1` and `3389` from the `.env` file
  are used in this command example.


- If the RDP client machine is not the same as the Docker host machine,

   - use a pre-configured SSH connection to forward the `$VRDP` port
     from the Docker host machine
     <span style="font-size: 0.95em">
     (the remote SSH server there should allow port forwarding)
     </span>

      ```shell
      # ssh -NL <Client-Local-IP>:<Client-Local-VRDP>:<$VRDPIP>:<$VRDP> <Docker-host-machine>
      ```

      for example:

      ```shell
      ssh -NL 127.1.2.3:3389:127.0.0.1:3389 192.168.0.1
      ```

   - connect the RDP client to the running guest OS

      ```shell
      # xfreerdp +v:<Client-Local-IP>:<Client-Local-VRDP> +video +clipboard
      ```

      substitute `<Client-Local-IP>` and `<Client-Local-VRDP>`
      for the values from the ssh command above, for example:

      ```shell
      xfreerdp +v:127.1.2.3:3389 +video +clipboard
      ```


<a name="Next-steps"></a>

## Next steps

- After initial tests confirm expected container behaiviuor,
  the `restart:` configuration entry in
  [`compose.yml`](https://github.com/o124/archlinux-virtualbox-host/blob/main/compose.yml)
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
  add the corresponding `ports:` entries in [`compose.yml`](https://github.com/o124/archlinux-virtualbox-host/blob/main/compose.yml),
  for example:

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

- download and install the
  [*Oracle VM VirtualBox Extension Pack*](https://www.virtualbox.org/manual/ch01.html#intro-installing)

- create user [`$VBUSR`](#VBUSR) with [`$VBUID`](#VBUID)

- create [`$VMDIR`](#VMDIR) and set its permissions

- set the timezone to [`$TZONE`](#TZONE)


The `entrypoint` script is described [below](#entrypoint).


The Dockerfile exposes mount points

  - [`$VMDIR`](#VMDIR) - for the guest machine folder

  - [`/home/$VBUSR`](#VBUSR) that holds the guest machine registration and other VirtualBox data


To run VirtualBox with lesser privileges,
the Dockerfile executes the
[`HEALTHCHECK`](https://docs.docker.com/engine/reference/builder/#healthcheck)
and
[`ENTRYPOINT`](https://docs.docker.com/engine/reference/builder/#entrypoint)
instructions as user [`$VBUSR`](#VBUSR).

The default `HEALTHCHECK` here only checks
[*MachineState*](https://www.virtualbox.org/sdkref/_virtual_box_8idl.html#a80b08f71210afe16038e904a656ed9eb)
reported by `vboxmanage showvminfo`.
If the `arch/dist/bin/healthcheck` file (not provided here) existed during the image generation,
it is executed instead and its exit code is reported to the `HEALTHCHECK` instruction.
This custom `healthcheck` can do more guest-relevant tests.
For example, it can check the availability of the services provided by the guest.


<a name="entrypoint"></a>

#### The *entrypoint* script.

The [`entrypoint`](https://github.com/o124/archlinux-virtualbox-host/blob/main/arch/dist/bin/entrypoint)
script controls the guest machine registration and life cycle.
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
(i.e. for instance when the guest OS stops),
then it waits for the remaining `VBoxSVC` and `VBoxXPCOMIPCD` processes to exit,
then the container stops running.

---

A safe guest OS shutdown can also be triggered by Docker host system events.

When the [Docker daemon](https://docs.docker.com/get-started/overview/#docker-architecture)
tries to stop the container, it sends the `TERM` signal to the container,
waits for [`$GRACE_PERIOD`](#GRACE_PERIOD) until it stops gracefully,
and if the container does not stop afterwards, kills it.

To allow a graceful shutdown of the guest OS withing the `$GRACE_PERIOD`,
the `entrypoint` script traps the `TERM` signal
and in response sends `acpipowerbutton` signal
(the default stop action) to the guest OS.
If the guest OS can handle the ACPI signal,
it should be able to shut down safely and let the `VBoxHeadless` process to
exit and let the container to stop normally.

The `INT` and `QUIT` signals are also handled and translated respectively
into the `savestate` and `poweroff` signals for the guest machine.
If the container does not respond properly to the `TERM` signal,
it can be forced to stop harder

```shell
sudo docker compose kill -s SIGQUIT
```

Also, setting [`stop_signal:`](https://docs.docker.com/compose/compose-file/#stop_signal)
in [`compose.yml`](https://github.com/o124/archlinux-virtualbox-host/blob/main/compose.yml)
to `SIGINT` or `SIGQUIT` respectively sets the default stop action for the guest machine
to `savestate` or `poweroff` when the container stops.

<a name="docker-service-overriding"></a>
Note, `docker.service` at the docker host requires
[overriding](https://wiki.archlinux.org/title/systemd#Drop-in_files) of the
[`TimeoutStopSec=`](https://www.freedesktop.org/software/systemd/man/systemd.service.html#TimeoutStopSec=)
value.
Otherwise, while stopping the `docker.service`, `systemd` will kill the docker daemon
and all running VirtualBox machines after the default 90 seconds
which could be not sufficient for some guest OS to stop properly.\
Here is an example drop-in
[file](https://github.com/o124/archlinux-virtualbox-host/blob/main/etc/systemd/system/docker.service.d/TimeoutStopSec.conf):

   ```unit file (systemd)
   # /etc/systemd/system/docker.service.d/TimeoutStopSec.conf

   [Service]
   # Set the appropriate value according to your guest OS needs
   TimeoutStopSec=5h
   ```


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

The _commands_ can be used for examples in the
[`command:`](https://docs.docker.com/compose/compose-file/#command)
configuration entry in [`compose.yml`](https://github.com/o124/archlinux-virtualbox-host/blob/main/compose.yml) or with
[`docker compose run`](https://docs.docker.com/engine/reference/commandline/compose_run/)
command

```shell
sudo docker compose run -it --rm --name base vbox state
```

Unrecognised commands are executed with `exec "$@"`
replacing thereby the `entrypoint` process.
This can be used to execute arbitrary commands in the container,
for example see [here](#interactive-bash-session).


<a name="variables"></a>

## Environment variables

The default values for the environment variables are defined in the
[`.env`](https://github.com/o124/archlinux-virtualbox-host/blob/main/.env)
file.

<a name="IMAGE_NAME"></a>
**$IMAGE_NAME** sets the image name
[[ref]](https://docs.docker.com/compose/compose-file/#image).

<a name="VBGUEST"></a>
**$VBGUEST** is a label used to construct the container name,
and `$COMPOSE_PROJECT_NAME`.
See how in [`.env`](https://github.com/o124/archlinux-virtualbox-host/blob/main/.env)
and [`compose.yml`](https://github.com/o124/archlinux-virtualbox-host/blob/main/compose.yml).
When starting multiple containers,
make sure that they use different `$VBGUEST` strings for them to get unique names.

<a name="COMPOSE_PROJECT_NAME"></a>
**$COMPOSE_PROJECT_NAME**
[[ref]](https://docs.docker.com/compose/reference/envvars/#compose_project_name)
is included in the folder names for the named volumes.

<a name="VMSRC"></a>
**$VMSRC** is mounted into [`$VMDIR`](#VMDIR) inside the container when it starts.
It should contain a complete VirtualBox machine to run, i.e.
a `*.vbox` file defining the machine and everything it refers to.
`$VMSRC` should have permissions allowing user
[`$VBUSR`](#VBUSR) with id [`$VBUID`](#VBUID)
inside the container to read and modify it.
The included helper script
[`set-perm-vmsrc`](https://github.com/o124/archlinux-virtualbox-host/blob/main/set-perm-vmsrc),
making use of the
[`.env`](https://github.com/o124/archlinux-virtualbox-host/blob/main/.env),
file can be executed in the Docker host system
to set the appropriate permissions on `$VMSRC` before starting the container.

<a name="VMDIR"></a>
**$VMDIR** is a directory inside the container
where [`$VMSRC`](#VMSRC) is to be mounted.
It is what `--basefolder` parameter of
[`vboxmanage createvm`](https://www.virtualbox.org/manual/ch08.html#vboxmanage-createvm)
command expects.
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
When starting multiple containers, make sure they use different `$VRDPIP:$VRDP` pairs.

<a name="VRDP"></a>
**$VRDP**
is a port number that should be set to the same *VRDP* port number
that is set in `.vbox` file, see
[here](#accessing-the-guest-os).
When starting multiple containers, make sure they use different `$VRDPIP:$VRDP` pairs.


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
