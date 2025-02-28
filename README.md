# The RHEL Image Mode demo
This project shows how to create a bootable container image with RHEL
Image Mode and then deploy that in several ways.

## Demo setup
Start with a minimal install of RHEL 9.4+ on either baremetal or as a
VM. Make sure this repository is on your RHEL 9.4+ instance using either
`git clone` or secure copy (`scp`).

During RHEL installation, configure a regular user with `sudo`
privileges on the host. These instructions assume that this repository
is cloned or copied to your user's home directory on the host
(e.g. `~/bootc-demo`). The below instructions use that assumption.

Login to the host using `ssh` and then run the following commands
to create an SSH keypair that you'll use later to access the edge
device. Even though you really should set a passphrase, skip that when
prompted to make the demo a little easier to run.

    cd ~/bootc-demo
    ssh-keygen -t rsa -f ~/.ssh/id_core
    ln -s ~/.ssh/id_core.pub .

Edit the `demo.conf` file and make sure the settings are correct. At a
minimum, you should adjust the credentials for simple content access and
the `CONTAINER_REPO` to match the fully qualified name for your bootable
container repository. Don't include the optional tag. The full list of
options in the `demo.conf` file are shown here.

| Option         | Description |
| -------------- | ----------- |
| SCA_USER       | Your username for Red Hat Simple Content Access |
| SCA_PASS       | Your password for Red Hat Simple Content Access |
| EDGE_USER      | The name of a user on the target edge device |
| EDGE_PASS      | The plaintext password for the user on the target edge device |
| BOOT_PASS      | A custom grub2 boot password for the target edge device |
| BOOT_ISO       | Minimal boot ISO used to create a custom ISO with additional kernel command line arguments and a custom kickstart file |
| CONTAINER_REPO | The fully qualified name for your bootable container repository |
| HOSTPORT       | The port mapped to the web server in the bootable container for testing |

Make sure to download the `BOOT_ISO` file (e.g. [rhel-9.4-x86_64-boot.iso](https://access.redhat.com/downloads/content/rhel)
to the local copy of this repository on your RHEL instance
(e.g. ~/bootc-demo). Run the following script to update the system.

    sudo ./register-and-update.sh
    sudo reboot

After the system reboots, run the following script to install container
and ISO image tools.

    cd ~/bootc-demo
    sudo ./config-bootc.sh

Login to Red Hat's container registry using your Red Hat customer portal
credentials and then pull the container image for the base bootable
container.

    podman login registry.redhat.io
    podman pull registry.redhat.io/rhel9/rhel-bootc:latest

The conversion container runs as root, so you'll need to login
as root to `registry.redhat.io` using your [Red Hat customer
portal](https://access.redhat.com) credentials to pull the tools to
transform into other image types.

    sudo podman login registry.redhat.io
    sudo podman pull registry.redhat.io/rhel9/bootc-image-builder

At this point, setup is complete.

## Demo
In this demo, we'll build a bootable container image, test it, and then
show several ways that it can be deployed.

### Build the operating system image
First, inspect the `Containerfile` and see that these are the same
familiar commands to tailor a container. The base container image contains
a pre-built bootable container image that can be tailored.

Run the following commands to create a new bootable container image by
layering onto the exising one.

    cd ~/bootc-demo
    . demo.conf
    podman build -f Containerfile -t $CONTAINER_REPO:v1

This build will take much less time than building from a blueprint file
with image builder. Once the bootable container is built, you can test
it by running it as an ordinary container with podman.

    podman run -d --rm --name lamp -p 8080:80 $CONTAINER_REPO:v1 /sbin/init

NB: If you are running containers within your bootable container
image, you'll need to elevate privileges to test "containers within
containers". Specifically, the above command would instead look like:

    sudo podman run -d --rm --name lamp -p 8080:80 \
        --privileged --cap-add SYS_ADMIN --security-opt unmask=all \
        $CONTAINER_REPO:v1 /sbin/init

The `/sbin/init` command launches systemd within the container to then
run all of the services. The container though is passing kernel calls
to the host kernel rather than using its own kernel.

Use `curl` to browse to the web server within the running container -OR-
simply use your browser to connect to the URL.

    curl -s http://localhost:8080 | grep -i 'rhel image mode'

With podman, you can also remote shell into the running container to
explore the filesystem contents and layout.

    podman exec -it lamp /bin/bash
    exit

When you're finished confirming that the bootable container is correct,
you can stop the running container with the command below. Because the
`podman run` command included the `--rm` option, the container will be
removed from the host's filesystem when it stops.

    podman stop lamp

Now that you have a tested bootable container image, push the repository
to your registry to make the image available to others.

    . demo.conf
    podman login $(echo $CONTAINER_REPO | cut -d/ -f1)
    podman push $CONTAINER_REPO:v1

Also, push this image with the `prod` tag since that's what we'll want
to run on our target edge devices.

    podman tag $CONTAINER_REPO:v1 $CONTAINER_REPO:prod
    podman push $CONTAINER_REPO:prod

Make sure this container repository is publicly accessible. You may need
to log into your registry using a browser to change the accessibility.

### Deploy the image using an ISO file
Run the following command to generate an installable ISO file for your
bootable container. This command prepares a kickstart file to pull
the bootable container image from the registry and install that to the
filesystem on the target system. This kickstart file is then injected
into the RHEL boot ISO you downloaded earlier. It's important to note
that the content for the target system is actually in the bootable
container image in the registry.

    sudo ./gen-iso.sh

The generated file is named `bootc-lamp.iso`. Use that file to boot a
physical edge device or virtual guest. Make sure this system is able to
access your public registry to pull down the bootable container image.

Test the deployment after the system reboots by browsing to the IP address
of the physical edge device or virtual guest. The address should resemble
`http://<IP-Address>`.

### Updating the image
The bootable container image can be updated by simply editing or creating
a new `Containerfile`, rebuilding the container image, and then pushing
the new bootable container image to the registry.

Let's create `v2` of our container by adding a new package `strace`. In
the same directory, create the file `NewContainerfile`.

    printf "FROM $CONTAINER_REPO:v1\nRUN dnf -y install strace" \
        > NewContainerfile

Build the new image and tag it as `v2`.

    podman build -f NewContainerfile -t $CONTAINER_REPO:v2

Test the new image by running it as a container locally.

    podman run -d --rm --name lamp -p 8080:80 $CONTAINER_REPO:v2
    podman exec -it lamp /bin/bash

Within the container, run the following command to confirm that the
`strace` executable was added.

    which strace

Exit the container and then stop it.

    exit
    podman stop lamp

Now that the container image has been updated and tested, push the image
to the registry.

    podman push $CONTAINER_REPO:v2

Tag the new container as `prod` and push that to the registry as well.

    podman tag $CONTAINER_REPO:v2 $CONTAINER_REPO:prod
    podman push $CONTAINER_REPO:prod

The edge system that was recenty installed has a
`bootc-fetch-apply-updates` systemd timer and service that periodically
checks the registry and then pulls updates to the bootable container. This
mechanism can be used to update the operating system, simply by pushing a
new image to the registry.

Instead of waiting for the timer, on the target system you can force an
update using the following command.

     sudo bootc update --apply --quiet

You can tailor the `bootc-fetch-apply-updates.timer`
to change the timing of how often this runs by copying
`/usr/lib/systemd/system/bootc-fetch-apply-updates.timer` to
`/etc/systemd/system` and then editing the file.

### Deploy the image by converting to a qcow2 disk image
Bootable container images can be converted to other disk image formats
using the bootc-image-builder tooling which is also containerized. In
this example, we'll convert the bootable container to a qcow2
filesystem image. Other image formats are available. Check the
[documentation](https://github.com/osbuild/bootc-image-builder#-image-types)
for what's currently supported.

No kickstart file is used for this conversion so a JSON configuration
file is supplied to the bootc-image-builder tooling to add a user to
the filesystem image. Create a configuration file to set the username,
hashed password, and SSH public key for a user to be added to the QCOW2
disk image.

    . demo.conf
    envsubst '$EDGE_USER $EDGE_HASH $SSH_PUB_KEY' \
        < config.json.orig > config.json

The following command has a lot of parameters. It passes the custom
JSON configuration to the bootc-image-builder tooling to create a QCOW2
filesystem image from the bootable container image we built earlier.

    sudo podman run --rm -it --privileged -v .:/output \
        -v ./config.json:/config.json --pull newer \
        registry.redhat.io/rhel9/bootc-image-builder \
        --type qcow2 --config /config.json \
        $CONTAINER_REPO:prod

You can now create a virtual guest using the QCOW2 filesystem image.

