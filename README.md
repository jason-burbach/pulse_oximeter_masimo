# Masimo Rad-8 Pulse Oximeter Data to InfluxDB

A simple **Python3** script to read serial data from a **Masimo Rad-8 Pulse Oximeter**. Samples are extracted
at a frequency of 1 sample/second, placed in JSON format and sent to a local **InfluxDB** instance. This
data is visualized using **Grafana**.

## Purpose

My daughter has a trach and is ventilator dependent. My wife and I need to watch her O2 percentages
and were only able to do so if we were sitting in front of the pulse oximeter. Now, using InfluxDB 
and Grafana running on a local server, we can see her specific oxygen and heartrate from anywhere in
our home.

## Long-term Goal

My daughter has intermittent desats which I do not fully understand. My hypothesis is that since
she also has a **VP Shunt** that it is related to the barometric pressure. This is why I am also
monitoring the barometric pressure, indoor temperature & humidity using Bosch's **BME280**. After
approximately 30-days, I will be exporting the pulse-oximeter and weather data to look for correlation
between her breathing and other external factors.

## Software Dependencies

1. influxdb         (Database Server - Named as such if using Raspbian or other Debian derivative)
2. python3-influxdb (Python 3.x - Client)
3. serial Package   (Python 3.x - Necessary for serial communication via RS232/DB-9)
4. grafana          (Dashboard frontend and server)
5. smbus2           (Python 3.x - Optional for weather data correlation)
6. bme280           (Python 3.x - Optional for weather data correlation)

## Hardware Dependencies

1. Masimo Rad-8 Pulse Oximeter (I have not tested other pulse oximeters)
2. Serial to USB               (Male RS232/DB-9 to USB)
3. Raspberry Pi Zero W         (Or practically any other flavor of computer. This was $9.00)
4. Bosch bme280 Temp, Humidity & Pressure Sensor (Optional if weather data is desired)

## Environment Setup & Deployment with uv

This project uses [uv](https://docs.astral.sh/uv/) to manage its Python interpreter, virtual environment,
and dependencies. The toolchain is pinned end to end for reproducibility:

- **`.python-version`** pins the interpreter to **Python 3.14**. uv installs this exact version itself (a
  standalone build) on any machine — the dev box and the Pi included — so it never depends on whatever
  system Python happens to be present.
- **`pyproject.toml`** declares the dependencies.
- **`uv.lock`** pins the exact resolved versions of every dependency for reproducible installs.

To change the interpreter version, run `uv python pin <version>` (this rewrites `.python-version`) followed
by `uv sync`.

### Install uv (dev machine and Raspberry Pi)

```sh
curl -LsSf https://astral.sh/uv/install.sh | sh
```

The standalone installer fetches the correct prebuilt binary automatically (aarch64 on 64-bit Raspberry
Pi OS, armv7 on 32-bit). Re-open the shell or run `source ~/.bashrc` so `uv` is on your `PATH`.

### Local development

First create the environment with uv, then activate it like any standard virtual environment:

```sh
uv sync                          # create .venv/ and install deps from uv.lock
uv sync --extra weather          # also install smbus2 + RPi.bme280 for bme_influx.py

source .venv/bin/activate        # activate the venv (uv creates a standard .venv/)
python pox_2_influx.py           # now `python` is the project interpreter
deactivate                       # exit the venv when done
```

Upgrade the dependencies by running

```shell
# Main Dependencies
uv pip compile --upgrade --no-emit-index-url pyproject.toml

# Weather dependencies
uv pip compile --upgrade --no-emit-index-url --extra weather pyproject.toml -o requirements-weather.txt
```

Use `uv sync` again whenever dependencies change, and `uv add <package>` to add one (it updates both
`pyproject.toml` and `uv.lock`).

### Deploying to the Raspberry Pi

After the code (including `uv.lock`) is synced to the Pi, install the locked dependencies:

```sh
uv sync --extra weather          # or plain `uv sync` if not using the BME280
```

For the cron-driven `graf_keep_going.sh`, call the venv's interpreter by its path directly — no
activation needed, which is exactly what you want in a non-interactive cron job:

```sh
./.venv/bin/python ./pox_2_influx.py &
```

## Grafana Dashboard for Pulse Oximeter Data w/ Inside Weather

![alt text](pulse_grafana_screenshot1.png)

## How to Use

These scripts are installed on a Raspberry Pi Zero W running Debian Buster Lite. The Pi is attached
to my daughters pulse oximeter which is sometimes mobile. Due to the pulse oximeter and pi being
mobile, I energize the pi using a cell phone battery charger. Because the charge is not infinite and I
do not want to hook the pi up to a monitor or ssh in every power cycle, I use a keep_going script. The
keep_going script has been setup to run every minute as a cronjob. If the keep_going script sees
pox_2_influx running in the background, nothing is done and keep_going will exit. If pox_2_influx is not
running, then it will be executed. Please change the path w/in graf_keep_going.sh to reflect the location
of pox_2_influx.py.

The bme_influx.py file as mentioned above is to capture the weather parameters from a Bosch BME280 sensor
also running on the Raspberry Pi Zero W. The sensor is connected to the i2c bus on port 1 and address 0x77.
This script does not need or require the keep_going script but is currently setup to run every minute via
a cronjob.

The two primary files are:

1. pox_2_influx.py
2. graf_keep_going.sh

Additional weather tracker using BME280:

1. bme_influx.py

## Credit Where Credit is Due

I was using a rather unimpressive web site hosted on my local network to review my daughters real-time
spO2 readings until I stumbled upon this repo: https://github.com/yahnatan/pulseox

I would like to thank the above individual for their work. I can now trend, decompose and thouroughly
nerd out on my daughters time-series, pulse oximeter data.
