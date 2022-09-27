+++ 
date = 2020-01-06
title = "Installing Podman remote client on macOS using vagrant"
description = "Installing Podman remote client on macOS using vagrant."
slug = "" 
tags = ["podman","macOS","vagrant"]
categories = []
externalLink = ""
series = []
socialShare=true
+++

I am installing podman as a remote client on macOS using Vagrant. I  will not cover the vagrant setup in this post.

## Podman remote client

Podman is the tool to start and manage containers. On macOS, we have to use a thin remote client that connects to an actual Podman process running on a Linux host.

Here are the main steps how to configure the remote client to work with a Linux host:

- Create a Linux machine using Vagrant
- Set key based ssh as root to the Linux host
- Install remote-client binary with Homebrew: brew cask install podman

Create a fedora vagrant box.

```bash
mkdir fedora-box && cd fedora-box
echo "Vagrant.configure("2") do |config|
  config.vm.box = "generic/fedora30"
  config.vm.hostname = "fedora30"
  config.vm.provider "virtualbox" do |v|
    v.memory = 1024
    v.cpus = 1
  end
end" >> Vagrantfile
vagrant up && vagrant ssh
```

Create new `ssh` keys on macOS and copy the newly generated public key.

```bash
ssh-keygen
cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3...
```

Add ssh keys copied earlier on the Linux host to `.ssh/authorized_keys`

```bash
echo "ssh-rsa AAAAB3..." >> /root/.ssh/authorized_keys
```

On the Linux host, install Podman and varlink socket. The remote client uses this to execute commands calling Podman’s API.

```bash
sudo dnf --enablerepo=updates-testing install podman libvarlink-util libvarlink
```

Install podman on macOS using Homebrew

```bash
brew cask install podman
```

Once podman is installed, create a connection parameters in `$HOME/.config/containers/podman-remote.conf`

```bash
cat <<EOF >$HOME/.config/containers/podman-remote.conf
[connections]
    [connections.host1]
    destination = "127.0.0.1"
    username = "root"
    default = true
    port = 2222
EOF

# With the remoting file configured, we can run podman simply as:
podman images
REPOSITORY   TAG   IMAGE ID   CREATED   SIZE
```

Verify running a container:

```bash
podman run --name tomcat -d docker.io/tomcat
Trying to pull docker.io/tomcat
....
d2e6db3c7....
```

Building images:

**Note:** The podman-remote.conf file seems to be ignored by the podman build command, so we have to add `--remote-host 127.0.0.1 --username root --port 2222` to each command

```bash
podman --remote-host 127.0.0.1 --username root --port 2222 build --tag mytag 
```
