# raspberry-box

A lightweight tool to customize a raspberry pi image on Linux.

Some notable features:

 * Setup wireless networks to connect to
 * Setup static or DHCP configuration
 * Install services or scripts to run at boot
 * Enable SSH
 * Disable image resizing at boot
 * Set user/root passwords

 And much more!

 ## Quickstart

**Install rbox from source**

Make sure you have the Go compiler installed first.

 ```shell
go get github.com/twitchyliquid64/raspberry-box/rbox
go build -o rbox github.com/twitchyliquid64/raspberry-box/rbox # Creates ./rbox
 ```

**Write out a config**

Save a file like this into your project directory (or any directory).

```python
# mypi.box
load('pi.lib', "pi")

# setup is called before build().
def setup(img):
    image = pi.load_img(img)
    return struct(image=image)

# build is called to actually build the image.
def build(setup):
    pi.configure_hostname(setup.image, 'my-pi')
    pi.enable_ssh(setup.image)
    pi.configure_static_ethernet(setup.image, address='192.168.1.5/24', router='192.168.1.1')
    pi.configure_pi_password(setup.image, password='whelp')
    pi.configure_wifi_network(setup.image, ssid='test', password='network')

    pi.run_on_boot(setup.image, 'custom-print', '/bin/echo Custom script started yo!!!!!!!!!!!')
```

**Run rbox**

```shell
cp 2019-07-10-raspbian-buster-lite.img mypi.img # Make a copy for customization
sudo ./rbox --img mypi.img --script mypi.box # Actually customize the image
```

## Config documentation

Configuration files are written in a python dialect called [starlark](https://github.com/bazelbuild/starlark).

### Overview

At the top of the file you should import any builtin libraries you need using the `load` function.

When you invoke `rbox` with your config, it will call two functions:

1. `setup(img)` - setup is called with the path to the image as the argument. This function should mount the image
   and perform any sanity checks.
2. `build(setup)` - build is called with the return value from `setup()`. You should put most of your configuration in here.

### Using pilib

*Don't forget to import pilib in your config! `load('pi.lib', "pi")`*

Shortcuts for performing basic tasks (like setting the hostname) are provided for you in `pilib`, so using them is the
simplest way to get started.

#### `load_img(<path>)`

This function loads as raspberry pi image at the specified path, checks its partitions, and mounts both partitions.
The return value is a structure with two fields:

1. `.ext4` - The ext4 partition (main system files)
2. `.fat` - The fat partition (kernel command line, boot partition, etc)

#### `configure_pi_hostname(<image>, <hostname>)`

This function sets a hostname on the given image. The first parameter should be the return value of `load_img()`.

#### `enable_ssh(<image>)`

This function creates the `ssh` file in the FAT partition, causing the sshd service to be enabled.
The first parameter should be the return value of `load_img()`.

#### `cmdline(<image>)`

This function returns the kernel command line as a string.
The first parameter should be the return value of `load_img()`.

#### `disable_resize(<image>)`

This function disables resizing of the SD card on first startup.
The first parameter should be the return value of `load_img()`.

#### `configure_pi_password(<image>, <password>)`

This function sets the password of the `pi` user. The password is saved in `/etc/shadow` in the usual fashion, with a randomly generated salt & using the SHA512 algorithm. The first parameter should be the return value of `load_img()`.

#### `configure_wifi_network(<image>, <ssid>, <wifi_password>)`

This function tells the wifi card to connect to the given network, using the ssid and password provided.
The first parameter should be the return value of `load_img()`.

This function only supports providing a single wifi-password combination: so multiple calls to `configure_wifi_network`
will overwrite the previous entry.

#### `configure_static_ethernet(<image>, <address>, <router IP>, <optional DNS server IP>)`

This function configures the ethernet port with a static IP address and default gateway. For example:

```python
pi.configure_static_ethernet(image, address='192.168.1.5/24', router='192.168.1.1')
```

Configures the ethernet port to use address `192.168.1.5`, on a `/24` (`255.255.255.0`) subnet, using `192.168.1.1` as the default
gateway.
If no DNS address is specified, `8.8.8.8` is used.

#### `configure_dynamic_ethernet(<image>, <optional lease seconds>, <optional hostname>)`

This function configures the ethernet port to request network configuration from the LAN.

#### `run_on_boot(<image>, <name>, <program string>, <optional username>, <optional groupname>)`

This function sets up a program to run at boot, under the provider user/group if one was provided (otherwise
  root is used).

The program string you provide should be a full path to the program, and all its arguments.

The name should be relatively unique, and only consist of lowercase letters and hyphens.

Under the hood, this method generates a systemd unit + service, and installs it in the raspberry Pi image as a requirement
to the `multi-user.target` target.
