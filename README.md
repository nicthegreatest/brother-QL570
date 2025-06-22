# Brother QL-570 Setup Guide for Fedora Linux
This guide provides a complete, reliable method for setting up a Brother QL-570 label printer on a modern Fedora Linux system (like Fedora 42 and later).

This method bypasses the official, outdated Brother drivers and uses a modern open-source tool to communicate directly with the printer. This solves the common issue where CUPS reports a print job as "Completed," but nothing actually prints.

## Step 1: Install System Dependencies
First, we need to install a few essential packages from the Fedora repositories that are required for the open-source driver to build and run correctly.

Open a terminal and run the following command:

sudo dnf install python3-pip python3-devel libusb1-devel ImageMagick

python3-pip: The package installer for Python, used to get the driver.

python3-devel: Development files needed to build some of the driver's components.

libusb1-devel: Development files for the library that communicates with USB devices.

ImageMagick: A powerful tool for creating and manipulating images, which we'll use to create test labels.

## Step 2: Install the Open-Source brother_ql Driver
Next, use pip to install the open-source Python driver. The --user flag installs it into your home directory, which avoids needing sudo for this step.

pip install --user --upgrade brother_ql

Step 3: Find Your Printer's USB ID
The system needs a unique way to identify the printer. Plug in the printer and turn it on, then run:

lsusb | grep Brother

You will see a line similar to this:
> Bus 001 Device 008: ID 04f9:2028 Brother Industries, Ltd QL-570 Label Printer

The important part is the ID: 04f9:2028. The first part (04f9) is the idVendor, and the second (2028) is the idProduct. We will use these in the next step.

## Step 4: Create a udev Rule for Permissions
By default, Linux prevents normal users from directly accessing hardware for security reasons. We will create a udev rule to grant specific permissions for the printer as soon as it's plugged in.

Create and open a new rule file with a text editor:

> sudo nano /etc/udev/rules.d/99-brother-ql-permissions.rules

Copy the following line and paste it into the editor. Ensure you use the idVendor and idProduct you found in Step 3.

> SUBSYSTEM=="usb", ATTR{idVendor}=="04f9", ATTR{idProduct}=="2028", GROUP="lp", MODE="0664"

This rule tells the system: when a USB device with this specific vendor/product ID is connected, assign it to the lp group and set its mode to allow read/write access for the owner and the group.

Save the file and exit the editor (Ctrl+O, Enter, Ctrl+X in nano).

## Step 5: Add Your User to the lp Group
Now that the device will belong to the lp group, you must add your user account to that group to gain access.

> sudo usermod -a -G lp <your_username>

(Replace <your_username> with your actual username)

## Step 6: Apply Changes and Reboot
To ensure all permission changes take full effect system-wide, the most reliable method is to perform a full reboot.

reboot

After rebooting, log back in. You can verify you are in the lp group by running the groups command. You should see lp in the list.

## Step 7: Test Printing from the Command Line
You are now ready to print directly.

Create a correctly-sized test image. For a 62x100mm label, the driver expects an image that is exactly 696x1109 pixels. Use ImageMagick to create one:

convert -size 696x1109 xc:white -fill black -draw "rectangle 100,100 596,300" ~/test_label.png

Run the print command. Use the idVendor and idProduct from Step 3.

brother_ql --backend pyusb --model QL-570 --printer usb://04f9:2028 print -l 62x100 ~/test_label.png

A label with a black rectangle should now print. You can ignore any post-print warnings like Insufficient amount of data received as long as the label prints successfully.

(Optional) Step 8: Create a Convenience Script
To make printing easier, you can create a simple script.

Create a file named print-label:

nano ~/print-label

Paste the following code into the file:




>#!/bin/bash
>
># ==============================================================================
># Automated Print-Label Script for Brother QL-570
># Version: 2.0
># Features:
>#   - Automatically detects and converts SVG files to print-ready PNGs.
>#   - Prints PNG files directly.
>#   - Uses a temporary file for conversions, which is cleaned up automatically.
># ==============================================================================
>
># --- Configuration ---
># The vendor and product ID of your printer, found with `lsusb`.
>PRINTER_ID="04f9:2028" 
># The pixel dimensions your printer expects for the label size.
># For 62x100mm, this is 696x1109.
>LABEL_DIMENSIONS="696x1109"
># The label code for the `brother_ql` tool.
>LABEL_SIZE_CODE="62x100"
>
># --- Script Logic ---
>
># 1. Check if a filename was provided as an argument.
>if [ -z "$1" ]; then
>  echo "Error: You must provide a path to an image file (PNG or SVG)."
>  echo "Usage: ./print-label <path_to_image>"
>  exit 1
>fi
>
>INPUT_FILE="$1"
># This will be the final file that gets sent to the printer.
>FILE_TO_PRINT=""
>
># 2. Create a temporary file that will be deleted when the script exits.
># This is where the converted SVG will be stored.
>TEMP_PNG_FILE=$(mktemp /tmp/print-label-XXXXXX.png) 
>trap 'rm -f "$TEMP_PNG_FILE"' EXIT 
>
># 3. Check if the input file is an SVG (case-insensitive check).
>if [[ "${INPUT_FILE,,}" == *.svg ]]; then
>  echo "SVG file detected. Converting to a print-ready PNG..."
>  
>  # Use ImageMagick's `magick` to handle the SVG.
>  # It resizes the SVG to fit, adds a white background, and sets the
>  # final canvas size to the exact dimensions the printer needs.
>  magick "$INPUT_FILE" -resize "$LABEL_DIMENSIONS" -background white -gravity center -extent "$LABEL_DIMENSIONS" "$TEMP_PNG_FILE"
>  
>  # Check if the conversion was successful.
>  if [ $? -ne 0 ]; then
>    echo "Error: ImageMagick SVG conversion failed."
>    exit 1
>  fi
>  
>  # Set the file to print to our newly created temporary PNG.
>  FILE_TO_PRINT="$TEMP_PNG_FILE"
>else
>  # If it's not an SVG, we assume it's already a print-ready PNG.
>  FILE_TO_PRINT="$INPUT_FILE"
>fi
>
># 4. Print the final image file.
>echo "Sending '$FILE_TO_PRINT' to the printer..."
>brother_ql --backend pyusb --model QL-570 --printer "usb://$PRINTER_ID" print -l "$LABEL_SIZE_CODE" "$FILE_TO_PRINT"
>
># The `trap` command will automatically delete the temporary file now.
>echo "Done."






Save and exit (Ctrl+O, Enter, Ctrl+X).

Make the script executable:

> chmod +x ~/print-label

Now you can print any compatible image simply by running: ./print-label /path/to/your/image.png
