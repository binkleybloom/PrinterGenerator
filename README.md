PrinterGenerator
================

This script will generate a ["nopkg"](https://groups.google.com/d/msg/munki-dev/hmfPZ7sgW6k/q6vIkPIPepoJ) style pkginfo file for [Munki](https://github.com/munki/munki/wiki) to install a printer.

See [Managing Printers with Munki](https://github.com/munki/munki/wiki/Managing-Printers-With-Munki) for more details.

## Usage

The script can either take arguments on the command line, or a CSV file containing a list of printers with all the necessary information.  

### Using a CSV file:

A template CSV file is provided to make it easy to generate multiple pkginfos in one run.  Pass the path to the csv file with `--csv`:

```
./print_generator.py --csv /path/to/printers.csv
```
*Note: if a CSV file is provided, all other command line arguments are ignored.*

The CSV file's columns should be pretty self-explanatory:

* Printer name: Name of the print queue
* Location: The "location" of the printer
* Display name: The visual name that shows up in the Printers & Scanners pane of the System Preferences, and in the print dialogue boxes.  Also used in the Munki pkginfo.
* Address: The IP or DNS address of the printer. The template uses the form: `lpr://ADDRESS`.  Change to another protocol in the template if necessary.
* Driver: Name of the driver file in /Library/Printers/PPDs/Contents/Resources/.
* Description: Used only in the Munki pkginfo.
* Required Driver pkg: Name of the required printer driver installation package - must exist in munki repo.
* Options: Any printer options that should be specified. These **must** be space-delimited key=value pairs, such as "HPOptionDuplexer=True OutputMode=normal".  **Do not use commas to separate the options, because this is a comma-separated values file.**

The CSV file is not sanity-checked for invalid entries or blank fields, so double check your file and test your pkginfos thoroughly.

### Command-line options:

A full description of usage is available with:

```
./print_generator.py -h
usage: print_generator.py [-h] [--printername PRINTERNAME]
						  [--driver DRIVER]
                          [--address ADDRESS]
                          [--location LOCATION]
                          [--displayname DISPLAYNAME]
                          [--desc DESC]
			  [--driverpkg MUNKIDRIVERPKG]
                          [--options [OPTIONS [OPTIONS ...]]]
                          [--version VERSION]
                          [--csv CSV]

Generate a Munki nopkg-style pkginfo for printer installation.
```

As in the above CSV section, the arguments are all the same:

* `--printername`: Name of the print queue. **Required.**
* `--driver`: Name of the driver file in /Library/Printers/PPDs/Contents/Resources/. **Required.**
* `--address`: The IP or DNS address of the printer. The template uses the form: `lpr://ADDRESS`.  Change to another protocol in the template if necessary. **Required.**
* `--location`: The "location" of the printer. If not provided, this will default to the value of `--printername`.
* `--displayname`: The visual name that shows up in the Printers & Scanners pane of the System Preferences, and in the print dialogue boxes.  Also used in the Munki pkginfo.  If not provided, this will default to the value of `--printername`.
* `--desc`: Used only in the Munki pkginfo. If not provided, will default to an empty string ("").
* `--driverpkg`: Used only in the Munki pkginfo. Will set a required install of specified driver installation pkg.
* `--options`: Any number of printer options that should be specified. These should be space-delimited key=value pairs, such as "HPOptionDuplexer=True OutputMode=normal".
* `--version`: The version number of the Munki pkginfo. Defaults to "1.0".

### Figuring out options:

Printer options can be determined by using `lpoptions` on an existing printer queue:  
`/usr/bin/lpoptions -p YourPrinterQueueName -l`  

Here's a snip of output:

```
OutputMode/Quality: high-speed-draft fast-normal *normal best highest
HPColorMode/Color: *colorsmart colorsync grayscale
ColorModel/Color Model: *RGB RGBW
HPPaperSource/Source: *Tray1
Resolution/Resolution: *300x300dpi 600x600dpi 1200x1200dpi
```

Options typically have the actual PPD option name on the left side of the /, and a display name (which is likely to show up in the printer settings dialogue boxes) on the right of the /.  The possible values for the printer are listed after the colon.  The current option is marked with an "*".  

Despite `lpoptions` using a "Name/Nice Name: Value *Value Value" format, the option must be specified more strictly:  
"HPColorMode=grayscale"

This is the format you must use when passing options to `--options`, or specifying them in the CSV file.  

*Note that `/usr/bin/lpoptions -l` without specifying a printer will list options for the default printer.*


### Example:
```
./print_generator.py --printername="MyPrinterQueue" \
		--driver="HP officejet 5500 series.ppd.gz" \
		--address="10.0.0.1" \
		--location="Tech Office" \
		--displayname="My Printer Queue" \
		--desc="Black and white printer in Tech Office" \
		--driverpkg="CanonDrivers" \
		--options="HPOptionDuplexer=True OutputMode=normal" \
		--version=1.5
```

The output pkginfo file generated:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>autoremove</key>
	<false/>
	<key>catalogs</key>
	<array>
		<string>testing</string>
	</array>
	<key>description</key>
	<string>Black and white printer in Tech Office</string>
	<key>display_name</key>
	<string>My Printer Queue</string>
	<key>installcheck_script</key>
	<string>#!/usr/bin/python
import subprocess
import sys

printerOptions = { "HPOptionDuplexer":"True OutputMode",  }

cmd = ['/usr/bin/lpoptions', '-p', 'MyPrinterQueue', '-l']
proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
(lpoptOut, lpoptErr) = proc.communicate()

if lpoptErr:
    sys.exit(0)

for option in lpoptOut.splitlines():
    for myOption in printerOptions.keys():
        optionName = option.split("/", 1)[0]
        optionValues = option.split("/",1)[1].split(":")[1].strip().split(" ")
        for opt in optionValues:
            if "*" in opt:
                actualOptionValue = opt.replace('*', '')
                break
        if optionName == myOption:
            if not printerOptions[myOption] == actualOptionValue:
                sys.exit(0)

sys.exit(1)</string>
	<key>installer_type</key>
	<string>nopkg</string>
	<key>minimum_os_version</key>
	<string>10.7.0</string>
	<key>name</key>
	<string>AddPrinter-MyPrinterQueue</string>
	<key>postinstall_script</key>
	<string>#!/usr/bin/python
import subprocess
import sys

# Populate these options if you want to set specific options for the printer. E.g. duplexing installed, etc.
printerOptions = { "HPOptionDuplexer":"True OutputMode",  }

cmd = [ '/usr/bin/lpadmin', '-x', 'MyPrinterQueue' ]
proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
(lpadminxOut, lpadminxErr) = proc.communicate()

# Install the printer
cmd = [ '/usr/sbin/lpadmin',
        '-p', 'MyPrinterQueue',
        '-L', 'Tech Office',
        '-D', 'DISPLAY_NAME',
        '-v', 'lpd://ADDRESS',
        '-P', '/Library/Printers/PPDs/Contents/Resources/HP officejet 5500 series.ppd.gz',
        '-E',
        '-o', 'printer-is-shared=false',
        '-o', 'printer-error-policy=abort-job' ]

for option in printerOptions.keys():
    cmd.append("-o")
    cmd.append(str(option) + "=" +  str(printerOptions[option]))

proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
(lpadminOut, lpadminErr) = proc.communicate()

if lpadminErr:
    print "Error: %s" % lpadminErr
    sys.exit(1)
print "Results: %s" % lpadminOut    
sys.exit(0)</string>
	<key>unattended_install</key>
	<true/>
	<key>uninstall_method</key>
	<string>uninstall_script</string>
	<key>uninstall_script</key>
	<string>#!/bin/bash
/usr/sbin/lpadmin -x MyPrinterQueue</string>
	<key>uninstallable</key>
	<true/>
	<key>requires</key>
	<array>
		<string>CanonDrivers</string>
	</array>
	<key>version</key>
	<string>1.5</string>
</dict>
</plist>
```
