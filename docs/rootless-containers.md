# (Experimental) Rootless containers in Garden

With the latest release of Garden it is now possible to create and run processes
in containers without requiring root privileges! This document details the various
components that enable this to work, as well as providing a step-by-step guide for installing
and configuring Garden to run as a non-root user.

**HUGE DISCLAIMER**: Garden's support for rootless containers is still very much 
a work-in-progress at the moment, and as such is subject to a number of known
limitations (see the end of this doc for details).

**Special Thanks**: A huge thanks to the [OCI community](https://www.opencontainers.org/) (and
especially [Aleksa Sarai](https://github.com/cyphar)) for their hard work and support
in making rootless containers a reality.

## Component Overview

Following is a brief overview of the components required to enable rootless containers
with Garden.

* `gdn` - an all-in-one, standalone version of [Garden-runC](https://github.com/cloudfoundry/garden-runc-release),
the container engine powering [Cloud Foundry](https://www.cloudfoundry.org/) and [Concourse CI](http://concourse.ci/).
* `grootfs` - a daemonless container image manager.
* `runc` - a CLI tool for spawning and running containers according to the OCI specification.

# Getting Started

The following documents the process of installing, configuring and running Garden
as a non-root user on an Ubuntu Xenial machine.

If you run into any issues along the way, feel free to chat to us on the
`#garden` channel of the [Cloud Foundry Slack](http://slack.cloudfoundry.org/).

## Prerequisites

* An Ubuntu Xenial machine (with kernel version 4.4+)

## Step 1: Install gdn and grootfs

The first step is to download and install `gdn` and `grootfs`. Note that `runc`
does not need to be installed separately as it is bundled together as part of the
`gdn` binary. The [install-rootless-gdn](../scripts/install-rootless-gdn) script
can be used to get started.

**NB**: The commands in Step 1 must be run as the root user. The rootless fun doesn't begin until step 2!

```
ubuntu@ubuntu-xenial:~$ sudo su -
root@ubuntu-xenial:~# curl "https://raw.githubusercontent.com/cloudfoundry/garden-runc-release/develop/scripts/install-rootless-gdn" | bash
```

This script will:

* Create a new, non-root user named `rootless`
* Install all required binaries to `/usr/local/bin/<binary name>`
* Configure a BTRFS filesystem at `/var/lib/grootfs/btrfs`
* Set permissions on dirs used by `gdn` and `grootfs`

Once the install script has completed, you'll need to run the `gdn setup` command:

```
root@ubuntu-xenial:~# gdn setup
```

This command is responsible for mounting cgroups and configuring iptables chains
(both of which still require root permissions at the moment).

## Step 2: Start the `gdn` server

**NB**: The commands in Step 2 must be run as the rootless user (created in Step 1).


## Networking
step1: build from cf-networking the binaries
git clone https://github.com/cloudfoundry-incubator/cf-networking-release.git
cd cf-networking-release
git submodule update --init
GOOS=linux GOARCH=amd64 go install garden-external-networker
GOOS=linux GOARCH=amd64 go install github.com/containernetworking/cni/plugins/main/bridge
GOOS=linux GOARCH=amd64 go install github.com/containernetworking/cni/plugins/ipam/host-local

transfer the binaries to a path dir -> /usr/local/bin

chmod u+s {garden-external-networker, bridge,host-local}

$ cat /tmp/file.json
{
"cni_plugin_dir": "/usr/local/bin",
"cni_config_dir": "/tmp/config",
"bind_mount_dir": "/var/vcap/data/garden-cni/container-netns",
"overlay_network" :"10.0.2.0/24",
"state_file" : "/var/vcap/data/garden-cni/external-networker-state.json",
"start_port" : 1000,
"total_ports" : 2000,
"iptables_lock_file" :"/var/vcap/data/garden-cni/iptables.lock",
"instance_address" : "1.2.3.4",
"iptables_asg_logging" : false
}


$ cat 30-bridge.conf
{
    "name": "mynet",
    "type": "bridge",
    "bridge": "mynet0",
    "isDefaultGateway": true,
    "forceAddress": false,
    "ipMasq": true,
    "hairpinMode": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.10.0.0/16"
    }
}


modify gdn not to do /etc/{hosts,resolve}

make sure iptables is in the $PATH
export PATH=$PATH:/sbin

$ gdn server   --bind-ip 0.0.0.0   --bind-port 7777   --image-plugin /usr/local/bin/grootfs   --image-plugin-extra-arg=--store   --image-plugin-extra-arg=/var/lib/grootfs/btrfs   --network-plugin /home/ubuntu/cf-networking-release/bin/garden-external-networker --network-plugin-extra-arg='--configFile=/tmp/file.json'  --skip-setup

inside the container ping 8.8.8.8 (iptable enable)


```
root@ubuntu-xenial:~# su - rootless
rootless@ubuntu-xenial:~$ gdn server \
  --bind-ip 0.0.0.0 \
  --bind-port 7777 \
  --image-plugin /usr/local/bin/grootfs \
  --image-plugin-extra-arg=--store \
  --image-plugin-extra-arg=/var/lib/grootfs/btrfs \
  --network-plugin /bin/true \
  --skip-setup
```

As shown above, `gdn` is configurable and extensible via plugins. At the moment `gdn` provides
a plugin interface for image and network management. The image plugin is fulfilled by `grootfs`
(which is also able to run without root privileges) but the network plugin is essentially a no op...
We'll be adding network support to rootless containers [shortly](https://www.pivotaltracker.com/story/show/141110133).

## Step 3: Enjoy rootless containers

**NB**: The commands from Step 3 onwards can be run as any user.

The `gaol` CLI (installed in Step 1) can be used to interact with Garden.
Containers can be created as follows:

```
ubuntu@ubuntu-xenial:~$ gaol create -n my-rootless-container -r docker:///busybox
my-rootless-container
```

And processes can be run in containers as follows:

```
ubuntu@ubuntu-xenial:~$ gaol run my-rootless-container -a -c "echo Hello Rootless :D"
Hello Rootless :D
ubuntu@ubuntu-xenial:~$ gaol run my-rootless-container -a -c "sh -c 'exit 13'"
ubuntu@ubuntu-xenial:~$ echo $?
13 # the exit code of the container process gets propagated
ubuntu@ubuntu-xenial:~$ gaol run my-rootless-container -c "sh -c 'while true; do echo cake && sleep 1; done'"
3b119c01-007c-4023-4a4e-65ef3629e647 # without the -a flag, gaol will detach from the process and print the process's ID
ubuntu@ubuntu-xenial:~$ gaol attach my-rootless-container -p 3b119c01-007c-4023-4a4e-65ef3629e647
cake
cake
cake
^C
```

And finally containers can be destroyed as follows:

```
ubuntu@ubuntu-xenial:~$ gaol destroy my-rootless-container
```

## Known Limitations

* There is currently no support for resource limiting
* Rootless containers do not have any networking (but will do relatively [soon](https://www.pivotaltracker.com/story/show/141110133))
* `gdn` cannot currently run as _any_ non-root user, it must be run as the `rootless` user
* You can only map 1 user into the container ([for now](https://www.pivotaltracker.com/story/show/130628237))
* Probably lots of other things as well
