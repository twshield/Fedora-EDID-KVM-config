# Fedora-EDID-KVM-config

## Issue

The DisplayPort specification requires that monitors provide an [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data), but inexpensive KVM switches like the
[IOGEAR 2-Port USB DisplayPort Cable KVM](https://www.iogear.com/product/GCS52DP/) do not provide EDID information to the inactive computer.  This can cause [display problems](https://docs.kernel.org/admin-guide/edid.html).

The following is a combination of things I found in various places on the internet and thought it would be good to have them all in one place.

You will need to do the following on all Linux computers connected to the KVM.  Reboot the computer after making the changes below.

Note that Nvidia GPU drivers can manage EDID files, see for example [Managing a Display EDID on windows](https://nvidia.custhelp.com/app/answers/detail/a_id/3569/~/managing-a-display-edid-on-windows) and there are xorg.conf options for Nvidea drivers to set the EDID as well.  As far as I know, the following also works with Nvidia GPUs but I have not tested it.

The solution for Linux is to use the kernel parameters:

### video=

see [modedb default video mode support](https://docs.kernel.org/fb/modedb.html)

### drm.edid_firmware=

see [Kernel Parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html?highlight=drm+edid_firmware)

## Setting Kernel Parameters:

Note that the following worked on Fedora 38 with AMD video cards and using X11, other distributions and combinations maybe different.  These parameters also support multiple monitors and video cards, but I'm not covering that here.  See the links above for more information on this.

Add the following to your `/etc/default/grub` file in the `GRUB_CMDLINE_LINUX=` line:
```
video=DP-1:3840x2160@30e drm.edid_firmware=edid/edid.bin
```

You will need to modify the line above to match your configuration.  Then in Fedora running `grub2-mkconfig` will update the options in the files in `/boot/loader/entries/`.  Or you can modify one of these by hand to test it out.  This example uses the same EDID for all ports, which I have simply named `edid.bin` in this example.  Putting `DP-1:` in front of the EDID filename will apply it to only that port.

I found that I did not need to use the `nomodeset` kernel parameter, and in fact it caused problems.

### getting the current display mode

I found that I did not get working virtual consoles unless I also specified the display mode to use.  The command `xrandr` will show you the modes and indicate which one you are using. For my monitor I used the mode `3840x2160@30` in the above.  Note that the trailing `e` in the video parameter is to enable the port, it is not part of the mode.

### getting video port in use 

The `DP-1` above is the video port that I have connected to my KVM (note that xrandr displays different names, don't use these).  Find your connected port by looking at the directories in `/sys/class/drm`, I have 
```
card1
card1-DP-1
card1-DP-2
card1-DVI-D-1
card1-HDMI-A-1
card1-HDMI-A-2
```
and doing `cat /sys/class/drm/card1-DP-1/status` shows that it is connected.

### getting an EDID file for your monitor 

Use the monitor-edid package command:
```
monitor-get-edid > edid.bin
```
This will get the EDID of the currently connected monitor.  I gave my edid file a name with the monitor model in it.

Put this file in `/lib/firmware/edid/`, you might need to make the edid subdirectory.

### Adding the EDID file to the initramfs

It turns out that the video module (amdgpu in my case) looks for the EDID file early in the boot process, so it needs to be in the initramfs file.
Create the file `/etc/dracut.conf.d/99-edid.conf` with the contents:
```
install_items+=" /lib/firmware/edid/edid.bin "
```
and then run `dracut -f` to add this to the currently running kernel's initramfs file.
You can check the contents of this file with `lsinitrd`  You can grep `dmesg` output for `edid` to check for errors related to loading this file after rebooting.

## Virtual Console font size

Not directly related to this, but I'll put it here anyways, is how to set the font size on virtual consoles.  With my 4k monitor and old eyes the default font size was too small.  So I put the following in `/etc/vconsole.conf`
```
FONT="LatGrkCyr-12x22"
```
and commented (with a leading `#`) out the FONT= line that was already there. Your choices for console fonts are in `/usr/lib/kbd/consolefonts/` in Fedora.


