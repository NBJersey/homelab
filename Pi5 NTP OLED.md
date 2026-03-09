
# NTP Server Dashboard: Waveshare 2.42" OLED (SPI)

This supplementary guide details the installation and configuration of a Waveshare 2.42" OLED (SKU: WAV-25743) to act as a physical Stratum 1 UTC display for the `ahierntp1` Raspberry Pi 5 server.

Because the I2C bus is saturated by the Uputronics RV3028 RTC, this display is configured to use **SPI**, ensuring zero bus contention and a high refresh rate for smooth millisecond updates.

---

## 1. Hardware Connections (SPI)

The Waveshare module uses a 7-pin GH1.25 cable. These wires must be connected to the Raspberry Pi 5's GPIO header. Since the Uputronics HAT sits on the header, you will connect these to the HAT's passthrough pins.

| OLED Pin | Wire Color (Typical) | Pi 5 GPIO Pin | Purpose |
| --- | --- | --- | --- |
| **VCC** | Red | **Pin 2 or 4** (5V) | Power (5V recommended for best contrast) |
| **GND** | Black | **Pin 6** (Ground) | Common Ground |
| **DIN** | Blue | **Pin 19** (GPIO 10) | SPI MOSI (Data In) |
| **CLK** | Yellow | **Pin 23** (GPIO 11) | SPI Clock |
| **CS** | Orange | **Pin 24** (GPIO 8) | SPI CE0 (Chip Select) |
| **DC** | Green | **Pin 22** (GPIO 25) | Data / Command Control |
| **RST** | White | **Pin 13** (GPIO 27) | Hardware Reset |

*Note: Ensure the 0Ω resistor on the back of the Waveshare PCB is soldered in the "SPI" position (this is usually the factory default).*

---

## 2. OS & SPI Configuration

Debian 13 (Trixie) disables SPI by default. You must enable it in the firmware to allow the Pi to communicate with the display.

1. **Edit the boot config:**
```bash
sudo nano /boot/firmware/config.txt

```


2. **Add or uncomment the following line:**
```text
dtparam=spi=on

```


3. **Reboot the system:**
```bash
sudo reboot

```


4. **Verify SPI is active:**
```bash
ls /dev/spi*

```


*You should see `/dev/spidev0.0` and `/dev/spidev0.1` listed.*

---

## 3. Python Environment & Dependencies

Debian 13 strictly enforces PEP 668, meaning you cannot install Python packages globally using `pip` without a virtual environment. We will set up an isolated environment for the display driver.

1. **Install system dependencies:**
```bash
sudo apt update
sudo apt install python3-venv python3-dev libfreetype6-dev libjpeg-dev build-essential libopenjp2-7 libtiff5 -y

```


2. **Create the Python Virtual Environment:**
```bash
mkdir -p ~/oled_dash
cd ~/oled_dash
python3 -m venv venv

```


3. **Activate and install the Luma libraries:**
```bash
source venv/bin/activate
pip install luma.oled luma.core

```



---

## 4. The Display Script

This script pulls the ultra-precise system time (disciplined by Chrony and the PPS pulse) and renders it on the SSD1309 OLED.

1. **Create the script:**
```bash
nano ~/oled_dash/ntp_display.py

```


2. **Paste the following Python code:**
```python
from luma.oled.device import ssd1309
from luma.core.interface.serial import spi
from luma.core.render import canvas
from PIL import ImageFont
import time
import sys

# Initialize SPI interface for the Waveshare 2.42" (SSD1309)
# We specify the exact GPIO pins used for Data/Command (25) and Reset (27)
serial = spi(device=0, port=0, gpio_DC=25, gpio_RST=27)
device = ssd1309(serial)

def main():
    print("Starting Stratum 1 OLED Display...")
    try:
        while True:
            with canvas(device) as draw:
                t = time.time()

                # Format Time
                str_time = time.strftime('%H:%M:%S', time.gmtime(t))
                millis = int((t % 1) * 1000)

                # Header
                draw.text((0, 0), "STRATUM 1 NTP - UTC", fill="white")
                draw.line((0, 12, 128, 12), fill="white")

                # Main Clock Array
                # Note: luma.oled uses default fonts unless specified. 
                # For a larger aesthetic, you can load a custom TTF font here.
                draw.text((15, 22), f"{str_time}.{millis:03d}", fill="white")

                # Cluster/Status Footer (Example for K3s)
                draw.line((0, 48, 128, 48), fill="white")
                draw.text((0, 52), "ahiernode: 4/4 Sync OK", fill="white")

            # 20Hz refresh rate for fluid millisecond updates
            time.sleep(0.05)

    except KeyboardInterrupt:
        # Clear display on exit
        device.clear()
        sys.exit(0)

if __name__ == "__main__":
    main()

```



---

## 5. Running as a Background Service

To ensure the display starts automatically when the Pi boots and runs continuously, create a systemd service.

1. **Create the service file:**
```bash
sudo nano /etc/systemd/system/oled-dash.service

```


2. **Add the following configuration:**
*(Ensure you replace `yourusername` with your actual Linux user, e.g., `ahierhomelab`)*
```text
[Unit]
Description=Stratum 1 OLED Dashboard
After=chrony.service network.target

[Service]
Type=simple
User=yourusername
WorkingDirectory=/home/yourusername/oled_dash
ExecStart=/home/yourusername/oled_dash/venv/bin/python ntp_display.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

```


3. **Enable and start the service:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable oled-dash.service
sudo systemctl start oled-dash.service

```



You can check the status of your new dashboard service at any time by running `systemctl status oled-dash.service`.

---

Would you like me to walk you through generating a custom large-scale font file (TTF) so the time takes up the full width of the OLED screen, rather than using the default `luma` text size?
