# How to Manually Patch the Ubuntu Kernel for Mercusys MA530 Bluetooth Adapter

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/34821b63-7de7-4e7b-ae49-aed5456e44be" />


This guide explains how to manually add support for the Mercusys MA530 USB Bluetooth dongle (RealTek rtl8761b) on Ubuntu. It resolves the issue where automated scripts fail due to Ubuntu's custom kernel naming conventions throwing 404 errors when trying to download mainline source code.

By downloading your specific Ubuntu kernel source and compiling only the required module, you avoid module conflicts and compilation errors.

---

## Step 1: Install Build Tools and Enable Source Repositories

First, tell Ubuntu to allow downloading source code and install the necessary compilation tools.

Open your terminal and run the following commands:

```bash
# Enable source code repositories (handles both new and older Ubuntu versions)
sudo sed -i 's/Types: deb/Types: deb deb-src/' /etc/apt/sources.list.d/ubuntu.sources 2>/dev/null || true
sudo sed -i '/^#\sdeb-src /s/^# *//' /etc/apt/sources.list 2>/dev/null || true
sudo apt update

# Install compilation tools and dependencies
sudo apt install build-essential linux-headers-$(uname -r) dpkg-dev flex bison libssl-dev libelf-dev zstd xz-utils
```

> **Note:** If `apt update` complains about missing source repositories, open the **"Software & Updates"** app, check the box for **"Source code"**, close it to let it reload, and re-run the install command.

---

## Step 2: Download Your Kernel Source

Create a working directory and download the exact source code for your currently running kernel:

```bash
mkdir -p ~/ma530_patch
cd ~/ma530_patch
apt-get source linux-image-unsigned-$(uname -r)
```

This command will extract a folder named `linux-<version>`.

---

## Step 3: Apply the Patch Manually

Navigate into the bluetooth drivers folder of the downloaded source. Using the wildcard `*` automatically enters the correct version folder:

```bash
cd ~/ma530_patch/linux-*/drivers/bluetooth/
```

Open the `btusb.c` file in the nano text editor:

```bash
nano btusb.c
```

1. Press `Ctrl + W` to open the search function.
2. Type `0x2ff8, 0xb011` and hit **Enter**.
3. It will jump to a line looking like this:
   ```c
   	{ USB_DEVICE(0x2ff8, 0xb011), .driver_info = BTUSB_REALTEK },
   
   	/* Additional Realtek 8761BUV Bluetooth devices */
   ```
4. **Right below** that line, add the code for your Mercusys MA530. Ensure it looks exactly like this:
   ```c
   	{ USB_DEVICE(0x2ff8, 0xb011), .driver_info = BTUSB_REALTEK },
   
   	/* Additional Realtek 8761BUV Bluetooth devices */
   	{ USB_DEVICE(0x2c4e, 0x0115), .driver_info = BTUSB_REALTEK |
   						     BTUSB_WIDEBAND_SPEECH },
   ```
5. Save the file by pressing `Ctrl + O`, hit `Enter` to confirm the filename, and exit nano with `Ctrl + X`.

---

## Step 4: Compile the Bluetooth Module

While still inside the `drivers/bluetooth/` directory, compile the module against your current system kernel headers by running:

```bash
make -C /lib/modules/$(uname -r)/build M=$(pwd) CONFIG_BT_HCIBTUSB=m modules
```

This will quickly compile a new `btusb.ko` file in your current folder.

---

## Step 5: Replace the Existing Driver

Modern Ubuntu versions compress their kernel modules (usually ending in `.ko.zst` or `.ko.xz`). The following script will:
1. Locate your current driver.
2. Back it up safely.
3. Optimize the new file size by stripping debug symbols.
4. Compress and overwrite the old driver with the correct compression format.

You can **copy and paste this entire block directly into your terminal** and press Enter:

```bash
# Find the exact path of your current driver
ORIG_MOD=$(modinfo -n btusb)

# Create a backup just in case
sudo cp $ORIG_MOD ${ORIG_MOD}.bak

# Strip debug symbols to reduce the new module's file size
strip --strip-debug btusb.ko

# Compress and install the new module based on your system's compression standard
if [[ $ORIG_MOD == *.zst ]]; then
    zstd -f -19 btusb.ko -o btusb.ko.zst
    sudo cp btusb.ko.zst $ORIG_MOD
elif [[ $ORIG_MOD == *.xz ]]; then
    xz -f btusb.ko
    sudo cp btusb.ko.xz $ORIG_MOD
else
    sudo cp btusb.ko $ORIG_MOD
fi
```

---

## Step 6: Reload the Driver

To apply the changes without restarting your computer:

1. **Unplug** the Mercusys MA530 USB adapter.
2. Run the following commands to reload the module:
   ```bash
   sudo rmmod btusb
   sudo modprobe btusb
   ```
3. **Plug your USB adapter back in.** Your Ubuntu system should immediately recognize it, and Bluetooth will now be fully operational.

---

## Important Maintenance Note

Whenever Ubuntu undergoes a kernel update (e.g., your system updates from kernel `6.8.0-38` to `6.8.0-40`), the custom driver will be overridden by the update. 

Keep the `~/ma530_patch` folder structure in mind. If your Bluetooth adapter stops working after a system update, simply repeat this process starting from **Step 2** to patch it for the newly installed kernel version.
