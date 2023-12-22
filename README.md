# ESPHome-devices
A collection of my own ESPHome stuff.

For now, an opinionated baseline config of how I think best to implement ESPHome for both homebrew and reflashed COTS devices.

## The short answer
1. Clone or otherwise copy the yaml files of this repo into your ESPHome folder (`/config/esphome`).
1. Edit as appropriate (at least `/base/common/wifi.yaml` but probably also the ota password in `/base/common/base-config.yaml`).
1. (If you happen to have a 'bootstrapped' device, e.g. that I gave you)
    1. In the ESPHome gui, 'install' the device, but choose the download option instead of OTA. If prompted, you want the 'old' / ESPHome Flasher file format. 
    1. Connect to the 'backup' AP that the device is providing since presumably can't connect to my wifi :)
    1. If it doesn't automagically, open http://192.168.4.1/
    1. Upload your newly minted firmware! The device will reboot and you should see it on your network within ~2 mins.

## The long answer, e.g. what I think you should know

First off, you'll want a way to edit some files outside of the ESPHome interface. Whether you use the Samba share, SCP/SSH, or even the File Editor addon is up to you, although personally I find the Samba addon the easiest. The Samba method will also conveniently allow you to edit the files with VSCode (and there's a plugin to do suggestions / documentation links; although I haven't yet been able to get it to do OTAs (it does support OTA but only when you're running VSCode from where you have ESPHome installed))

Once you have the ESPHome addon installed, you'll note that (nb: the way this presently works is under some debate in the HA world) there is a `esphome` folder under `/config`. In here, any `.yaml` files you create will show up in the ESPHome UI as devices, conversely, any devices you create in ESPHome will have their configuration files generated & stored here.

ESPHome conveniently supports several methods of including content from multiple sources (files or Git), each with some use cases.

Take the following example skeleton configuration:

```
esphome:
  name: example-device
  friendly_name: Example Device

esp32:
  board: esp32dev

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "7osfA/15w6tJu21tycrbZA3IEugdw2SROr4yF9Pv7ks="

ota:
  password: "bb19f04d6f69863569c15922010b10b9"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Example-Device Fallback Hotspot"
    password: "BL9y0CdwwCb3"

captive_portal:
```

The ESPHome documentation can of course unpack it for you, but it's fairly obvious to see that if you end up having any number of devices, you're going to end up with some significant duplication. Gross.

The OG way ESPHome supported 'merging' files, was using the `<<: !include` syntax. You might have a 'base' configuration file, `base.yaml` containing:

```
# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "7osfA/15w6tJu21tycrbZA3IEugdw2SROr4yF9Pv7ks="

ota:
  password: "bb19f04d6f69863569c15922010b10b9"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Example-Device Fallback Hotspot"
    password: "BL9y0CdwwCb3"

captive_portal:
```

And then in your `Random-device-1.yaml` you could thusly:

```
esphome:
  name: example-device
  friendly_name: Example Device

esp32:
  board: esp32dev

<<: !include base.yaml
```

Right off the hop, there are a couple of problems with this approach:
1. All your devices are going to end up having the same fallback hotspot SSID
1. The placement of `<<: !include base.yaml` is crucial as ESPHome is literally going to blindly dump the contents of `base.yaml` in that exact location in `Random-device-1.yaml`
1. YAML keys can only be defined once, so you couldn't for example easily override the `wifi:` section in any way

Through the magic of popular FOSS, some new options have since been added that allow us to address these issues in very clean ways. The first is by having a 'smarter' inclusion mechanism using the `packages:` syntax. Instead of blindly merging the files, ESPHome will instead thoughtfully 'merge' your configuration, so that you could for example instead have `Random-device-2.yaml`:

```
esphome:
  name: example-device-2
  friendly_name: Example Device 2

esp32:
  board: esp32dev

packages:
    unimportant_label: !include base.yaml

wifi:
  ap:
    ssid: "Example Device 2 Fallback Hotspot"
```

And it'd be smart enough to not only merge the two instances of the `wifi:` key, but in this case, use the more specific SSID we've provided in `Random-device-2.yaml`.

> Also, to avoid ESPHome assuming `base.yaml` is itself a device, you can either prefix it's filename with a period, or put it in a subfolder. Personally I like the latter approach; that's what I've used here.

But wait, there's more! Wouldn't it be nice if we could somehow even handle this fallback AP situation better? Enter the `substitutions:` syntax:

```
substitutions:
  devicename: example-device-3
  friendly_devicename: Example Device 3

wifi:
  ap:
    ssid: "${friendly_devicename} Fallback Hotspot"
```

And as you'd guess, the SSID will nicely match things up. Very fortunately (not sure if the `substitutions:` key needs to occur before the inclusions?) you can combine these two mechanisms to do some really nice stuff: this is how the actual example files here are setup.
