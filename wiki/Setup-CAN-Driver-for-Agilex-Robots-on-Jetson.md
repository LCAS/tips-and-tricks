# Fix Missing `gs_usb` CAN Driver on Jetson

This guide explains how to build and install the missing `gs_usb` kernel module on a Jetson, so USB-CAN adapters using the `gs_usb` driver appear as `can0`, `can1`, etc.

Tested scenario: Jetson running a `tegra` kernel such as:

```bash
5.15.148-tegra
```

---

## 1. Check the running kernel

```bash
uname -r
```

Example output:

```bash
5.15.148-tegra
```

Do **not** hard-code desktop Ubuntu kernel versions such as:

```bash
5.15.0-164-generic
```

Always use:

```bash
$(uname -r)
```

---

## 2. Install required packages

```bash
sudo apt update
sudo apt install -y build-essential wget can-utils
```

Install the Jetson kernel headers:

```bash
sudo apt install -y nvidia-l4t-kernel-headers
```

Check that the kernel build directory exists:

```bash
ls -l /lib/modules/$(uname -r)/build
```

If this path does not exist, the kernel headers/build tree are still missing or not correctly installed.

---

## 3. Create a build directory

```bash
mkdir -p ~/can_driver
cd ~/can_driver
```

---

## 4. Download the `gs_usb` driver source

```bash
wget https://raw.githubusercontent.com/torvalds/linux/v5.15/drivers/net/can/usb/gs_usb.c
```

---

## 5. Create the Makefile

```bash
cat <<'EOF' > Makefile
obj-m += gs_usb.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
EOF
```

Important: this uses the currently running Jetson kernel automatically.

---

## 6. Compile the module

```bash
make
```

After successful compilation, check that the module exists:

```bash
ls -lh gs_usb.ko
```

Check that it was built for the current kernel:

```bash
modinfo ./gs_usb.ko | grep vermagic
uname -r
```

The `vermagic` output should contain your current kernel version, for example:

```bash
5.15.148-tegra
```

---

## 7. Install the module

Create the target module directory if it does not already exist:

```bash
sudo mkdir -p /lib/modules/$(uname -r)/kernel/drivers/net/can/usb/
```

Copy the module:

```bash
sudo cp gs_usb.ko /lib/modules/$(uname -r)/kernel/drivers/net/can/usb/
```

Update module dependencies:

```bash
sudo depmod -a
```

Load the module:

```bash
sudo modprobe gs_usb
```

---

## 8. Make the module load automatically on boot

Add `gs_usb` to `/etc/modules`:

```bash
echo "gs_usb" | sudo tee -a /etc/modules
```

Be careful: the correct file is:

```bash
/etc/modules
```

Do **not** accidentally write to:

```bash
/etc/moduleses
```

If that typo file was created, remove it:

```bash
sudo rm -f /etc/moduleses
```

---

## 9. Confirm the module is loaded

```bash
lsmod | grep gs_usb
modinfo gs_usb
dmesg | grep -i gs_usb
```

---

## 10. Plug in the USB-CAN adapter and check CAN interfaces

```bash
ip link show | grep can
```

You should see interfaces such as:

```bash
can0
can1
```

For more detail:

```bash
ip -details link show can0
ip -details link show can1
```

---

## 11. Configure CAN bitrate

If the interface is already up, bring it down first.

For `can0`:

```bash
sudo ip link set can0 down
sudo ip link set can0 type can bitrate 500000
sudo ip link set can0 up
```

For `can1`:

```bash
sudo ip link set can1 down
sudo ip link set can1 type can bitrate 500000
sudo ip link set can1 up
```

Check status:

```bash
ip -details link show can0
ip -details link show can1
```

Expected status should include something like:

```bash
can state ERROR-ACTIVE
bitrate 500000
```

---

## 12. Test CAN communication

Open one terminal and listen on `can0`:

```bash
candump can0
```

Open another terminal and send from `can1`:

```bash
cansend can1 123#DEADBEEF
```

If the adapters are connected correctly, the frame should appear in the `candump` terminal.

---

## 13. Common errors and fixes

### Error: wrong build path

If you see:

```bash
/lib/modules/5.15.0-164-generic/build: No such file or directory
```

You are using the wrong kernel path. Replace hard-coded kernel versions with:

```bash
/lib/modules/$(uname -r)/build
```

---

### Error: missing build directory

If you see:

```bash
/lib/modules/5.15.148-tegra/build: No such file or directory
```

Install the Jetson kernel headers:

```bash
sudo apt install -y nvidia-l4t-kernel-headers
```

Then check again:

```bash
ls -l /lib/modules/$(uname -r)/build
```

---

### Error: target copy path is not a directory

If you see:

```bash
cp: cannot create regular file '/lib/modules/.../kernel/drivers/net/can/usb/': Not a directory
```

Create the directory first:

```bash
sudo mkdir -p /lib/modules/$(uname -r)/kernel/drivers/net/can/usb/
```

Then copy again:

```bash
sudo cp gs_usb.ko /lib/modules/$(uname -r)/kernel/drivers/net/can/usb/
```

---

### Error: device or resource busy

If you see:

```bash
RTNETLINK answers: Device or resource busy
```

The CAN interface is probably already up. Bring it down before changing bitrate:

```bash
sudo ip link set can1 down
sudo ip link set can1 type can bitrate 500000
sudo ip link set can1 up
```

---

## 14. Useful final check commands

```bash
uname -r
lsmod | grep gs_usb
modinfo gs_usb
ip -details link show can0
ip -details link show can1
dmesg | grep -i gs_usb
```

---

## Notes

- Use this only when the Jetson kernel does not already include the `gs_usb` module.
- This module must match the running kernel version.
- If the Jetson kernel is updated, you may need to rebuild and reinstall the module.
- For most robot CAN buses, `500000` bitrate is common, but confirm your device requirements.
