Raspberry Pi 5 Stratum 1 NTP Server with Uputronics GPS-RTC
This guide details the end-to-end process for building a high-precision, Stratum 1 NTP server using a Raspberry Pi 5 and a Uputronics GPS-RTC HAT. This setup is specifically tuned for Debian 13 (Trixie) running the 6.12 kernel, serving a high-availability Kubernetes (K3s) cluster on a local 192.168.50.0/24 network.
1. Hardware Overview
 * SBC: Raspberry Pi 5
 * HAT: Uputronics GPS-RTC (featuring the RV3028 RTC chip)
 * Display (Optional): Waveshare 2.42" OLED (SKU: WAV-25743) driven via SPI
 * Network Environment: Unifi Cloud Gateway routing the 192.168.50.0/24 subnet.
2. Base OS Preparation
Start with a fresh installation of Debian 13 (Trixie).
Set Hostname
Establish the server's identity on the network:
sudo hostnamectl set-hostname ahierntp1

Update the local loopback address:
sudo nano /etc/hosts

Change 127.0.1.1 raspberrypi to 127.0.1.1 ahierntp1. Reboot to apply.
Install Required Packages
sudo apt update && sudo apt upgrade -y
sudo apt install gpsd gpsd-clients chrony pps-tools python3 python3-pip -y

3. Hardware Configuration (/boot/firmware/config.txt)
The Raspberry Pi 5 requires specific overlay mappings for the UART and the internal PMIC RTC to prevent conflicts with the Uputronics HAT.
Open the boot config:
sudo nano /boot/firmware/config.txt

Append the following configuration:
# 1. Enable UART 0 for GPS NMEA Data on Pi 5
dtparam=uart0-pi5=on

# 2. Configure PPS on GPIO 18
dtoverlay=pps-gpio,gpiopin=18

# 3. Configure the Uputronics RV3028 RTC and disable internal Pi 5 RTC
dtparam=i2c_arm=on
dtoverlay=i2c-rtc,rv3028,wakeup-source
dtparam=rtc_pi5=off

# 4. Enable SPI (If using the Waveshare 2.42" OLED)
dtparam=spi=on

Reboot the system after saving.
4. Permissions & Service Setup
In Debian Trixie, the daemons must have direct access to the character devices (/dev/ttyAMA0 and /dev/pps0).
Add the service users to the dialout group:
sudo usermod -a -G dialout gpsd
sudo usermod -a -G dialout _chrony

5. GPS Daemon (gpsd) Configuration
Configure gpsd to start automatically and point to the hardware UART, allowing it to automatically detect the linked PPS device.
Open the default configuration:
sudo nano /etc/default/gpsd

Replace the contents with:
START_DAEMON="true"
USBAUTO="false"
DEVICES="/dev/ttyAMA0 /dev/pps0"
GPSD_OPTIONS="-n"
GPSD_SOCKET="/var/run/gpsd.sock"

Restart the service:
sudo systemctl enable gpsd
sudo systemctl restart gpsd

6. Chrony (NTP) Configuration
This configuration bridges the ultra-precise PPS hardware pulse with the NMEA serial data for the "seconds" context.
Open the Chrony configuration:
sudo nano /etc/chrony/chrony.conf

Replace the contents with the following:
# 1. Hardware Clock - The Pulse (PPS)
# 'prefer' makes this the absolute authority. 'mintel -2' speeds up locking on Pi 5.
refclock PPS /dev/pps0 lock NMEA precision 1e-7 refID PPS prefer mintel -2

# 2. Hardware Clock - The Data (NMEA via GPSD SHM)
# Offset tuned for 115200 baud serial delay to align with the PPS pulse.
refclock SHM 0 delay 0.2 offset 0.053 refID NMEA noselect

# 3. Network Fallback
pool uk.pool.ntp.org iburst

# 4. Network Access
# Allow K3s nodes to sync
allow 192.168.50.0/24

# 5. Stratum 1 Persistence
# Serve time even if GPS lock is temporarily lost (relies on RV3028 RTC)
local stratum 1 weight 100

# Standard Housekeeping
driftfile /var/lib/chrony/chrony.drift
makestep 1.0 3
rtcsync
log tracking measurements statistics
logdir /var/log/chrony
maxupdateskew 100.0

Apply the configuration and force an initial step:
sudo systemctl restart chrony
sudo chronyc makestep

7. Verification Tools
Checking Chrony Sync Status
Run the following to verify Chrony is locking onto the PPS signal:
watch -n 2 chronyc sources -v

Expected Output: A * next to the PPS source and a Reach of 377.
Checking Hardware Jitter (Python Script)
Save this as pps_verify.py to directly poll the kernel's PPS discipline without interpretation overhead:
import time
import os
import sys

PPS_PATH = "/sys/class/pps/pps0/assert"

def get_pps_data():
    try:
        with open(PPS_PATH, "r") as f:
            line = f.read().strip()
            ts_str, sequence = line.split("#")
            sec_part, nano_part = ts_str.split(".")
            pps_timestamp_ns = (int(sec_part) * 1_000_000_000) + int(nano_part)
            return pps_timestamp_ns, int(sequence)
    except FileNotFoundError:
        print(f"Error: {PPS_PATH} not found.")
        sys.exit(1)

def main():
    print("Format: [Sequence] | Offset (μs) | Jitter (μs)")
    last_seq = -1
    last_offset = 0
    
    while True:
        pps_ns, seq = get_pps_data()
        if seq != last_seq:
            sys_ns = time.time_ns()
            offset_ns = sys_ns - pps_ns
            offset_us = offset_ns / 1000
            jitter_us = abs(offset_us - last_offset) if last_seq != -1 else 0
            
            print(f"[{seq:5}] | Offset: {offset_us:8.2f} μs | Jitter: {jitter_us:6.2f} μs")
            last_seq = seq
            last_offset = offset_us
            
        time.sleep(0.1)

if __name__ == "__main__":
    main()

Run with: sudo python3 pps_verify.py
8. Client Node Configuration (CentOS Stream 9 / K3s)
To ensure high-availability cluster nodes stay perfectly synchronized, configure their local Chrony daemons to prefer the new Stratum 1 server.
Edit /etc/chrony.conf on the cluster nodes:
# Primary Stratum 1 Source
server 192.168.50.5 iburst prefer

# Public Fallback
pool uk.pool.ntp.org iburst

driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
maxupdateskew 100.0
log tracking measurements statistics
logdir /var/log/chrony

Restart chronyd: sudo systemctl restart chronyd.
