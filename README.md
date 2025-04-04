# <p align="center">Truman - **I**ntelligent **T**ruma **A**doption
### <p align="center">Take control of your Truma - Using a Raspberry Pi &amp; LIN adapter


<p align="center">This binary eliminates the need for the original control panel or any further devices between your RPI and your Truma (apart from the LIN adapter).

<p align="center">It connects to a MQTT broker and will report the current status of the heater.

<p align="center">Control commands are also sent to the heater using MQTT.

# Disclaimer

This project is a hobbyist effort and is not affiliated with or endorsed by Truma in any way.
All trademarks and brand names are the property of their respective owners.

You're free to use, modify, and share this software under the terms of the GNU General Public License v3.0 (GPL-3.0).

That said, please keep in mind:

- You’re using this at your own risk.

- It might void your warranty if you hook it up to real hardware.

- We do our best, but there's no guarantee it won’t misbehave, break something, or confuse your heater.

- Always make sure you know what you're doing — and when in doubt, unplug it!

Have fun, hack responsibly, and feel free to contribute!

# Installing:

Make sure you have the needed packages installed:
```sudo apt update && sudo apt install -y libmosquitto1 libssl3 libstdc++6```

Enable the serial UART port in the ```sudo raspi-config```  → Interface Options → Serial

Make sure your user is added to the dialout group: ```sudo usermod -aG dialout $USER```

---

An .env file is mandatory for the script to start, it could look like this:

```
## mqtt broker
MQTT_HOST=localhost
## mqtt broker port
MQTT_PORT=1883
## optional username
MQTT_USERNAME=
## optional password
MQTT_PASSWORD=
## topic that is used to control truma
MQTT_TOPIC=truma/control
## topic that truma will report values on
MQTT_TOPIC_REPORT=truma/report
## delay between UART/MQTT processing loops (microseconds, 10000 = 10ms)
SLEEP_DURATION_US=10000
## delay between mqtt updates in seconds, prevents spamming the broker
MQTT_PUBLISH_INTERVAL=10
## UART port
UART_DEVICE=/dev/ttyAMA5
## baudrate
UART_BAUDRATE=9600
```

You can use the example .env file provided in the repo, or create your own. To quickly get started, run the following command to download both the .env and the precompiled binary:

```wget -O .env https://raw.githubusercontent.com/vincentgloss/trumanita/main/.env && wget -O LINsenddirect https://raw.githubusercontent.com/vincentgloss/trumanita/main/LINsenddirect```

Then make the binary executable: ```chmod +x LINsenddirect```

# Usage:

Start the binary with ```./LINsenddirect``` or if you want a different .env file use ```ENV_FILE=myCustom.env ./LINsenddirect```
Add flags ```-d``` or ```--debug``` for more output.

You might hear some relays clicking in the Truma heater when the binary is initialized and connection is established.
During runtime mqtt reports should come in every 10s (or whatever you set the publish interval to).

It is advised to keep the original control panel around, as the binary does not state any error codes, only that (if) an error has ocurred. You'll need to refer to the original panel to read the error code.

Stop the process with ```sudo pkill -f LINsenddirect``` if needed/desired.

## 🛰️ MQTT Control – Overview

### 📦 Full Configuration (preferred, JSON payload)

Default topic:  **truma/control/truma_config**

**Required fields:**
`heater`, `boiler`, `temp`, `fan`, `energymix`, `mode`, `mode2`


### 📦 Payload Examples

<table>
<tr>
  <td>

<strong>Boiler eco + Heater on Gas only</strong>

```json
{
  "boiler": "eco",
  "energymix": "gas",
  "fan": "eco",
  "heater": "on",
  "mode": "gas",
  "mode2": "gas",
  "temp": 22
}
```

  </td>
  <td>

<strong>Boiler Hot electric only</strong>

```json
{
  "boiler": "hot",
  "energymix": "electric",
  "fan": 5,
  "heater": "off",
  "mode": "mix2",
  "mode2": "electric",
  "temp": 25
}
```

  </td>
  <td>

<strong>Boiler boost + Heater on</strong>

```json
{
  "boiler": "boost",
  "energymix": "mix1",
  "fan": "high",
  "heater": "on",
  "mode": "mix1",
  "mode2": "gas",
  "temp": 29
}
```

  </td>
</tr>
</table>

**Please pay special attention to the three mode settings: `energymix`, `mode`, and `mode2`. Refer to your heater’s manual to understand how different combinations of these settings affect its behavior.**

All seven values have to be set in the payload, see possible values in the table below.

### 🔧 Example Commands (mosquitto_pub/sub)

Send a config (heter on, fully electric (900W, needs 230V connection!), ambient temp 25°C):
``` 
mosquitto_pub -h localhost -t truma/control/truma_config -m '{"heater":"on","boiler":"off","temp":25,"fan":"low","energymix":"electric","mode":"mix1","mode2":"electric"}'
```
Subscribe to the report topic:
```
mosquitto_sub -h localhost -t truma/report/#
```
---


### ⚡ Single Commands (topic + raw payload)

| Topic                    | Accepted Values                                              |
|--------------------------|--------------------------------------------------------------|
| `truma/control/boiler`   | `off`, `eco` (40°C), `hot` (60°C), `boost`                   |
| `truma/control/energymix`| `gas`, `electric`, `mix1`, `mix2`                            |
| `truma/control/fan`      | `off`, `eco`, `low`, `medium`, `high`, `max`, or `0–10`      |
| `truma/control/heater`   | `on`, `true`, `1` or `off`, `false`, `0`                     |
| `truma/control/mode`     | `gas`, `mix1` (900W), `mix2` (1800W)                         |
| `truma/control/mode2`    | `gas`, `electric`, `mix`                                     |
| `truma/control/temp`     | Integer `5–29` (°C)                                          |
| `truma/control/reset`    | `true` (reset to default values)                             |

**Reset command sets the following defaults:**
- `heater = off`
- `boiler = off`
- `temp = 20`
- `fan = 0`
- `energymix = gas`
- `mode = gas`
- `mode2 = gas`

---

### 📡 MQTT Feedback

**Default report prefix:**  
truma/report/...


| Field            | Description                                             |
|------------------|---------------------------------------------------------|
| `room_temp`      | Float string – current room temperature (°C)            |
| `water_temp`     | Float string – current boiler water temperature (°C)    |
| `voltage`        | Float string – system voltage (V)                       |
| `heating_state`  | String – current heater status                          |
| `boiler_mode`    | String – current boiler mode                            |
| `error_state`    | String – e.g., `Normal`, `Error`                        |
| `truma_config`   | JSON string – echo of the last applied configuration (truma_config only)    |


## ⚠️ Known Issues

The status messages are inaccurate! The ambient and water temperatures as well as the voltage readings are correct, but some of the other reported states are hard to interpret and may not make much sense. So please don’t rely too heavily on them.

The `error_state` field will either show `"Normal"` (no error) or `"Error"`. If you see `"Error"`, you'll need to check the actual error code on the original Truma display. Alternatively, restarting the script will reset everything to default values — this often helps and acts as a kind of soft reset.

Note: It's possible to send a combination of settings to the heater that results in an error or warning. Pay particular attention to the mode settings — the three mode-related fields (`energymix`, `mode`, and `mode2`) need to be compatible.

For example, setting `energymix` to `electric` (which requires 230V) and `mode`/`mode2` to `gas` will not work and may cause an error.

---

# Tested devices:

| Device                  | Status |
|-------------------------|--------|
| Truma Combi 4 E         |   ✅   |
| Truma Combi 6 E         |   ✅   |


---
<p align="center">Truma ❤️ Anita