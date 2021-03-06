# How to create a 1280x1080 (among others) scaled resolutions for your non-standard monitor

Ultrawide monitors with PBP (Picture by Picture) are becoming more common these days. It allows you to split the screen and connect multiple sources to your monitor (e.g. 2 computers at the same time)

The problem is that the resolution is typically not supported out of the box (E.g. a 2560x1080 monitor, when split into 2 will require a 1280x1080 resolution, which is not standard in Mac OS as of Sierra)

### Creating custom resolutions

**First**, we have to reboot in rootless mode, and disable the Integrity Protection, so we can create/override the screen file settings:

1. Reboot the Mac and hold down Command + R keys simultaneously when it restarts, this will boot OS X into Recovery Mode
1. When the “OS X Utilities” screen appears, pull down the ‘Utilities’ menu at the top of the screen instead, and choose “Terminal”
1. Type the following command into the terminal then hit return:
```
csrutil disable; reboot
```

The Mac will then reboot itself automatically, just let it boot up as normal. [Source](http://osxdaily.com/2015/10/05/disable-rootless-system-integrity-protection-mac-os-x/)


**Second**, we need to find out your display configuration settings

1. Connect your non-compliant monitor and switch into the PBP mode
1. Open a terminal
1. Run this command:
 ```
  ioreg -lw0 | grep IODisplayPrefsKey
 ```

You'll get something like these results:
```
    | |   | |         "IODisplayPrefsKey" = "IOService:/AppleACPIPlatformExpert/PCI0@0/AppleACPIPCI/IGPU@2/AppleIntelFramebuffer@0/display0/AppleBacklightDisplay-610-a034"
    | |   | | |       "IODisplayPrefsKey" = "IOService:/AppleACPIPlatformExpert/PCI0@0/AppleACPIPCI/IGPU@2/AppleIntelFramebuffer@1/display0/AppleDisplay-1e6d-59f1"
```

One of the settings is for your Macbook laptop monitor (`AppleBacklightDisplay-610-a034`), and the other is for the connected display (`AppleDisplay-1e6d-59f1`). The naming follows a convention: The first part e.g. `1e6d` is the hexadecimal version of the **DisplayVendorID**, while the second one `59f1` is the **DisplayProductID**.

Go ahead and get the decimal values for those. You can use something like this site: http://www.binaryhexconverter.com/hex-to-decimal-converter to do it online.

```
1e6d (Display Vendor ID) = 7789
59f1 (Display Product ID) = 23025
```

With this info at hand, let's go and modify (or create if it doesn't exist) the file that will hold our customn settings

**Third**, create/modify the DisplayProduct file.

From the terminal or finder, navigate to `/System/Library/Displays/Contents/Resources/Overrides/DisplayVendorID-xxxx`, where `xxxx` is the hex value of the Display Vendor ID from above (e.g. `DisplayVendorID-1e6d`)

Then, open a text editor as an admin (or get ready to be prompted for your password upo saving later). Open the file named `DisplayVendorProductID-yyyy`, where `yyyy` is the Display Product ID from above (e.g. `DisplayProductID-59f1`). Make sure there is no file extension. If this file doesn't exist, create it.

As of MacOS Catalina, `/System/Library/Displays/Contents/Resources/Overrides/` is Not writable, so it is necesary to create the corresponding `DisplayVendorID-xxxx/DisplayVendorProductID-yyyy` on `/Library/Displays/Contents/Resources/Overrides/` (e.g. `/Library/Displays/Contents/Resources/Overrides/DisplayVendorID-1e6d/DisplayProductID-59f1`)


**Fourth**, enter the DisplayProduct ID info.

Other than the DisplayProductID and the DisplayVendorID (that are now in decimal instead of hex), that should be self-explanatory, the other things here are:

- DisplayProductName, which can be any name,
- IODisplayEDID - Data, which I copied from another file in the same directory (e.g. `DisplayProductID-76db') because I noticed **it shared the same DisplayVendorID** for the monitor I was trying to configure (most likely because this is the non-PBP version of the driver)
- dspc - data, which I got from somebody that used SwitchResX :p

Here is the final file:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>DisplayProductID</key>
	<integer>23025</integer>
	<key>DisplayProductName</key>
	<string>SS Settings - LG ULTRAWIDE</string>
	<key>DisplayVendorID</key>
	<integer>7789</integer>
	<key>IODisplayEDID</key>
 <data>AP///////wAebdt2AQEBATIXAQSlUCJ4nsqVplVOoSYPUFQhCABxQIGAgcCpwLMA0cCBAAEB53xwoNCgKVAwIDoAIE8xAAAanWdwoNCgIlAwIDoAIE8xAAAaAAAA/QA4PR5aIAAKICAgICAgAAAA/AAzNFVNOTUKICAgICAgAVECAxFxIwkGB0QQBAMBgwEAAAI6gBhxOC1AWCxFACBPMQAAHn5IAOCgOB9AQEA6ACBPMQAAGAEdAHJR0B4gbihVACBPMQAAHowK0Iog4C0QED6WACBPMQAAGGs+uFBgoClQCCC4BCBPMQAAGp89cKDQoBVQMCA6ACBPMQAAGgAAcg==</data>
	<key>dspc</key>
	<array>
		<data>
		KScA4FA4H0BAQDoAAAAAAAAY
		</data>
	</array>
</dict>
</plist>
```

Alternatively, if you don't feel very good at using the `dspc` key, and instead just want to have custom resolutions, the way is to use the `scaled-resolutions` key. Just replace the dspc section with something like this:

```
  <key>scale-resolutions</key>
  <array>
    <data>
    AAAFAAAABDgAAAAB
    </data>
  </array>
```

That weird data value (AAAFAA...) is generated by converting the decimal values of the resolution into hexadecimal, into binary, into base64.

If it's too confusing, no worries, here is a little bash that you can do to get the values:
`echo $(printf '%.8x%.8x%.8x' WIDTH HEIGHT 1) | xxd -r -p | base64`

e.g. (for a 1280x1080)
`echo $(printf '%.8x%.8x%.8x' 1280 1080 1) | xxd -r -p | base64`

Then just plug the result into the file and voila!

PS. In case you want to "decode" the string, it is something like this (needs to be optimized):
1. E.g. for the data value AAAFAAAABDgAAAAB, in bash, `echo AAAFAAAABDgAAAAB | base64 --decode |  xxd`
2. Take the result (e.g. `0000 0500 0000 0438 0000 0001`), and run them through the [hexadecimal to decimal converter](http://www.binaryhexconverter.com/hex-to-decimal-converter) in groups of 8 numbers. E.g. 0000 0500, 0000 0438, 0000 0001
3. This should give you 1280, 1080 and 1, which is the screen resolution (1280x1080)

PS2. Don't forget to turn on the integrity protection `csrutil enable`

#### References
* [xxd info](https://linux.die.net/man/1/xxd)


 
## To Get the second monitor IODisplayEDID directly from ioreg 
`ioreg -lw0 -r -c "IODisplayConnect" -d 2 | grep IODisplayEDID | awk -F "<" '{print $2}' | awk -F ">" '{print $1}' | head -2 | tail -1 | xxd -r -p | base64`

Depending on the string that comes up it will tell you the manufacturer of the display. It will not display in plain text.
Assuming you have a Retina display, a string with LP in it means it's an LG.
`ioreg -lw0 | grep \"EDID\" | sed "/[^<]*</s///" | xxd -p -r | strings -6`
