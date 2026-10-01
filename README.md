# SolLimit: Complete Guide (30 inverters, ModbusPal, HES simulator)

SolLimit is the ESP32 firmware for the NSO inverter monitoring and control modem (ESP-IDF v6.1, C++). This guide takes you from parts on the desk to a full test:

- **ModbusPal** on your PC simulates **30 inverters** on RS-485.
- A **HES simulator** on your PC plays the NSO back end: an HTTPS registration API, a test PKI, and a Mosquitto MQTT broker (MQTT v5, TLS 1.3, client certificates).
- You send commands with a free MQTT client (`mosquitto_pub`, or **MQTTX** if you prefer a GUI) and watch the inverter registers change in ModbusPal.

Follow the parts in order. Every command is for **Windows PowerShell**.

---

## Part 0. What the firmware does (requirements map)

| Req | What the firmware does | Where |
|---|---|---|
| FR-1.1 | Modbus RTU master, polls every configured inverter (up to 32) | `curtail.cpp`, `modbus.cpp` |
| FR-1.2 | One RS-485 daisy chain. 1200 m is a wiring rule (Part 2.4) | hardware |
| FR-1.3 | One MQTT stream per inverter: `{prefix}/telemetry/{serial}` | `curtail.cpp` |
| FR-2.1–2.3 | Before its first MQTT connection: HTTPS POST of device ID, firmware version, hardware revision, ICCID, MSISDN, site parameters and a **CSR**. The answer carries the topic prefix and the client certificate signed by the PKI | `hes.cpp` |
| FR-2.4 | Key, certificate and prefix are kept in NVS. The **private key is created on the device and never leaves it**. Encrypting NVS is a production step (Part 13) | `hes.cpp` |
| FR-3.1 | Commissioning portal on the device's own Wi-Fi AP. Phones on the AP get internet through the uplink (NAPT) | `portal.cpp`, `main.cpp` |
| FR-3.2 | After activation the portal is closed for good | `portal.cpp` |
| FR-3.3 | Hold **Button 1 (GPIO18) for 10 s** | `portal.cpp` |
| FR-3.4 | After activation the button only sends a request to NSO. The portal opens only when an operator approves that request over MQTT | `hes.cpp` |
| FR-4.1 | MQTT **v5** over **TLS 1.3** (the device refuses older versions) on 8883 | `hes.cpp` |
| FR-4.2 | Mutual TLS with the certificate from registration | `hes.cpp` |
| FR-4.3 | Telemetry interval 15–30 s (portal setting, default 30) | `curtail.cpp` |
| FR-4.4 | Last Will `{"state":"offline"}`, retained, on `{prefix}/status` | `hes.cpp` |
| FR-5.1 | `task_id`, `active_power`, `reactive_power`, `expiration` (+ optional `inverters` list) | `curtail.cpp` |
| FR-5.2 | Every accepted `task_id` is stored in NVS (last 64). A repeat is rejected, even after a reboot | `curtail.cpp` |
| FR-5.3 | Setpoint written with FC06, read back. Telemetry reports the value the inverter actually holds | `curtail.cpp` |
| FR-5.4 | On expiry: back to **100 % active, 0 % reactive**. Running commands survive a reboot | `curtail.cpp` |
| FR-6.1/6.2 | NTP at boot, after every uplink reconnect, and every hour | `main.cpp` |
| FR-7.1–7.5 | One 40-byte record per inverter every 30 s, in a circular flash log. Sequential writing is the wear levelling | `datalog.cpp` |
| FR-7.6 | Records not delivered are re-sent with `"backfill":true` when MQTT comes back | `curtail.cpp` |

**Limit you must know (FR-7.2).** The 4 MB board has room for about **50,000 records**. That is about **14 hours** of data for 30 inverters (or 7 days for 2 inverters). Seven days for 30 inverters needs about 24 MB, so it needs an external flash chip (for example a W25Q256, 32 MB). See Part 13.

---

## Part 1. Parts and software

### 1.1 Hardware you have

| Item | Used for |
|---|---|
| ESP32 dev board (NodeMCU, WROOM-32, 4 MB) | The modem |
| AIR780E 4G module + SIM with a data plan + antenna | 4G uplink, ICCID, signal strength |
| 5–24 V DC supply, at least 2 A, for the 4G module | Module power (never power it from the ESP32) |
| MAX485 RS-485 module | ESP32 side of the RS-485 bus |
| USB to RS-485 converter | PC side of the bus (ModbusPal talks through it) |
| 2 push buttons (6x6 tactile) | Button 1 = portal. Button 2 = spare |
| Jumper wires, a twisted pair for A/B | |

### 1.2 Software to install (one time)

1. **ESP-IDF v6.1.** You have it at `D:\esp\v6.1\esp-idf`. Each new PowerShell window needs:
   ```powershell
   . C:\Espressif\tools\Microsoft.v6.1.PowerShell_profile.ps1
   ```
2. **Python 3** from python.org (you have 3.14), plus one package:
   ```powershell
   py -m pip install cryptography
   ```
3. **Java 8 or newer** (you have Java 21). Check with `java -version`.
4. **ModbusPal 1.6c**: download `ModbusPal.jar` from https://sourceforge.net/projects/modbuspal/ and put it in `D:\Tools\ModbusPal\`.
5. **Mosquitto 2.x** (broker + `mosquitto_pub` / `mosquitto_sub` clients): https://mosquitto.org/download/ → *mosquitto-2.x.x-install-windows-x64.exe*. Install it with the defaults (`C:\Program Files\mosquitto`).
6. *(Optional)* **MQTTX**, a free GUI MQTT client: https://mqttx.app
7. **Driver for the USB-RS485 converter** (CH340, FTDI or CP210x, depending on the chip). After plugging it in, open Device Manager → *Ports (COM & LPT)* and note its COM number (for example `COM4`). The ESP32 board has its own COM port (for example `COM5`).

---

## Part 2. Wiring (from "ESP Based NSO.pdf")

### 2.1 Pin table

| ESP32 pin | Goes to | Notes |
|---|---|---|
| GPIO17 (TX2) | AIR780E **RX** | UART2, 115200 8N1 |
| GPIO16 (RX2) | AIR780E **TX** | |
| 3.3V | AIR780E UART header **VCC** | Sets the module's UART logic to 3.3 V. It does **not** power the module |
| GPIO23 | AIR780E **Reset** | Active low. The firmware pulls it low for 400 ms, then releases it. Never driven high |
| — | AIR780E **USB Boot** | Leave unconnected (it forces firmware-update mode) |
| GPIO27 | MAX485 **DI** | RS-485 #1, UART1 |
| GPIO26 | MAX485 **RO** | |
| GPIO25 | MAX485 **DE and RE** (tie them together) | The UART driver switches transmit/receive |
| GPIO18 | Button 1 → GND | Portal button, internal pull-up |
| GPIO19 | Button 2 → GND | Spare, not used by the firmware |
| GND | AIR780E GND, its power supply GND, MAX485 GND, converter GND | **One common ground for everything** |

Not used: **RS-485 #2** (UART0 on GPIO33/32/4). You have one RS-485 module, and UART0 is the USB log you need for testing. All 30 inverters share RS-485 #1.

### 2.2 Wiring picture

```
                    5-24 V / 2 A supply
                          |  (+)  (-)----------------------------+
                    +-----+------------+                          |
                    |  AIR780E 4G      |  antenna, SIM            |
                    |  VCC  TX  RX RST |                          |
                    +---+---+---+---+--+                          |
                        |   |   |   |      GND -------------------+
    ESP32 3.3V ---------+   |   |   |                             |
    ESP32 GPIO16 <----------+   |   |                             |
    ESP32 GPIO17 -------------->+   |                             |
    ESP32 GPIO23 ------------------>+                             |
                                                                  |
    ESP32 GPIO27 ----> DI  +----------+                           |
    ESP32 GPIO26 <---- RO  |  MAX485  |  A ====twisted pair==== A  |  USB-RS485  ==USB==> PC (ModbusPal)
    ESP32 GPIO25 ----> DE+RE          |  B ==================== B  |  converter
    ESP32 3.3V   ----> VCC +----------+                           |
                                                                  |
    ESP32 GPIO18 ---[Button 1]--- GND                             |
    ESP32 GPIO19 ---[Button 2]--- GND                             |
    ESP32 GND -----------------------------------------------------+ (also MAX485 GND, converter GND)
```

### 2.3 Practical notes

- **TX always goes to RX.** If the module never answers `AT`, swap GPIO16/17 first.
- **Power the MAX485 module from 3.3 V.** At 5 V its RO pin outputs 5 V, which can damage GPIO26. If your module will not work at 3.3 V, power it from 5 V and put a divider on RO (1 kΩ in series, 2 kΩ to GND). A MAX3485 module is a native 3.3 V part.
- Some RS-485 modules switch direction automatically and have no DE/RE pins. Then leave GPIO25 unconnected.
- If no inverter ever answers, **swap A and B**. Some converters label them D+/D-, or 485+/485-.
- 6x6 buttons: wire across the diagonal (opposite corners) to always get the switched pair.

### 2.4 Field wiring (FR-1.2, for the real site)

- Use a daisy chain (one cable from inverter to inverter), not a star. Total length up to 1200 m at 9600 baud.
- Use shielded twisted pair, and put a **120 Ω** resistor across A/B at **both ends** of the chain only. Keep stubs short.
- 30 inverters plus the modem is 31 loads, inside the 32 unit-load limit of standard transceivers.

---

## Part 3. HES simulator on your PC (do this first)

The firmware has the simulator's CA certificate built in, so the simulator must exist before you build.

### 3.1 Make the test PKI and the broker config

```powershell
cd D:\Dinuk\itZorix\NSO\POC\SolLimit
py tools\hes_sim.py init
```

It prints something like:

```
Server certificate is valid for: 192.168.1.4, localhost, 127.0.0.1
CA copied to main/certs/hes_ca.pem  ->  rebuild the firmware (idf.py build)
Portal values:  registration URL  https://192.168.1.4:8443/api/v1/devices/register
                MQTT broker       mqtts://192.168.1.4:8883
```

**Write down the two portal values.** `192.168.1.4` is your PC's address on the LAN.

- This has already been run once on this PC (for 192.168.1.4). If it says `tools/hes/pki already exists`, you can keep it.
- If your PC's IP changes, run `py tools\hes_sim.py init --force --host <new IP>`, then rebuild, then `idf.py erase-flash` and flash again (the old device certificate no longer matches the new CA).
- Tip: reserve the PC's IP in your router (DHCP reservation) so it never changes.

What `init` created in `tools\hes\`:

| File | What it is |
|---|---|
| `pki\ca.crt`, `pki\ca.key` | The test "NSO PKI" CA. It signs everything |
| `pki\server.crt/.key` | Certificate for the broker and the API (valid for the IPs above) |
| `pki\operator.crt/.key` | Client certificate for you, the "NSO operator" (MQTTX / mosquitto_pub) |
| `mosquitto.conf` | Broker: port 8883, **TLS 1.3 only**, **client certificate required**, ACL |
| `acl.conf` | Each device may only use `nso/v1/<its id>/#`. The operator may use `nso/v1/#` |
| `commands\*.json` | Ready-made command payloads for the tests in Part 8 |

### 3.2 Open the firewall (admin PowerShell, one time)

```powershell
New-NetFirewallRule -DisplayName "SolLimit HES sim" -Direction Inbound -Protocol TCP -LocalPort 8443,8883 -Action Allow -Profile Any
```

### 3.3 Start the three simulator windows

Open **three** PowerShell windows. Leave them running.

**Window A: the broker**

```powershell
& "C:\Program Files\mosquitto\mosquitto.exe" -c D:\Dinuk\itZorix\NSO\POC\SolLimit\tools\hes\mosquitto.conf
```

You should see `Opening ipv4 listen socket on port 8883`.

**Window B: the registration API**

```powershell
cd D:\Dinuk\itZorix\NSO\POC\SolLimit
py tools\hes_sim.py serve
```

**Window C: watch everything the devices send**

```powershell
$P = "D:\Dinuk\itZorix\NSO\POC\SolLimit\tools\hes\pki"
$TLS = @("-h","192.168.1.4","-p","8883","--cafile","$P\ca.crt","--cert","$P\operator.crt","--key","$P\operator.key","-V","mqttv5")
& "C:\Program Files\mosquitto\mosquitto_sub.exe" @TLS -t "nso/v1/#" -v
```

(Use your own IP instead of 192.168.1.4.) It prints nothing until the device connects.

### 3.4 (Optional) MQTTX instead of mosquitto_sub/pub

New connection:

| Field | Value |
|---|---|
| Host | `mqtts://` `192.168.1.4` |
| Port | `8883` |
| Client ID | `hes-operator-mqttx` |
| MQTT Version | `5.0` |
| SSL/TLS | on, **SSL Secure** on, Certificate: **Self signed** |
| CA File | `tools\hes\pki\ca.crt` |
| Client Certificate File | `tools\hes\pki\operator.crt` |
| Client key File | `tools\hes\pki\operator.key` |

Connect, then subscribe to `nso/v1/#`. To send a command, publish to the topics in Part 8 with QoS 1 and paste the JSON from `tools\hes\commands\`.

---

## Part 4. ModbusPal with 30 simulated inverters

### 4.1 Start ModbusPal with serial support

ModbusPal needs the RXTX serial library. It is inside `ModbusPal.jar`, so extract it once:

```powershell
cd D:\Tools\ModbusPal
tar -xf ModbusPal.jar rxtx
```

Start ModbusPal (always this way, or the COM port list stays empty):

```powershell
cd D:\Tools\ModbusPal
java "-Djava.library.path=rxtx\win-x64" -cp "ModbusPal.jar;rxtx\win-x64\RXTXcomm.jar" modbuspal.main.ModbusPalGui
```

### 4.2 Load the 30-inverter project

1. Click **Load** and open `D:\Dinuk\itZorix\NSO\POC\SolLimit\tools\modbuspal\SolLimit-30-inverters.xmpp`.
2. You see 30 slaves, `INV-001` (address 1) to `INV-030` (address 30).
3. On the **Serial** tab, pick the **USB-RS485 converter's COM port**, not the ESP32's. Set 9600 baud, parity **None**, 1 stop bit. The project already has these.
4. Click **Run**. The little tilted "Modbus" label flashes whenever a request arrives.
5. Click the **eye** button of a slave to see its registers live.

### 4.3 Register map (every simulated inverter)

All are holding registers (read with FC03, written with FC06). Address 0 in ModbusPal is Modbus address 0.

| Reg | Name | Unit | INV-001 value | Written by SolLimit? |
|---|---|---|---|---|
| 0 | `limit_pc` | active power limit, % (0–100) | 100 | **yes** (active_power) |
| 1 | `q_set_pc` | reactive power setpoint, %, signed (-20 shows as 65516) | 0 | **yes** (reactive_power) |
| 2 | `p_kw x100` | active power, 0.01 kW | 2100 = 21.00 kW | no |
| 3 | `q_kvar x100` | reactive power, 0.01 kVAr | 10 = 0.10 kVAr | no |
| 4, 5 | `e_day_wh` high/low | daily energy, Wh (32-bit) | 0, 41234 = 41.234 kWh | no |
| 6, 7, 8 | `va/vb/vc x10` | phase voltage, 0.1 V | 2301 = 230.1 V | no |
| 9, 10, 11 | `ia/ib/ic x100` | phase current, 0.01 A | 3043 = 30.43 A | no |
| 12 | `alarms` | bitmask | 0 (INV-007 has 4) | no |

Each inverter has slightly different values (power is 20 + address kW), so you can tell them apart in the telemetry. You can type new values into ModbusPal while it runs, and the next telemetry shows them.

A real inverter uses its own register map: change the `R_*` numbers and the scaling in `poll_one()` (`main/curtail.cpp`) from its Modbus document.

---

## Part 5. Build and flash the firmware

```powershell
. C:\Espressif\tools\Microsoft.v6.1.PowerShell_profile.ps1
cd D:\Dinuk\itZorix\NSO\POC\SolLimit
idf.py build
idf.py -p COM5 -b 115200 erase-flash        # first time with this firmware: wipes the old settings
idf.py -p COM5 -b 115200 flash monitor
```

- Use your ESP32's COM port. This board fails at 460800 baud, so always pass `-b 115200`.
- `erase-flash` is needed once, because the partition table changed (the data log replaced the OTA slot).
- Leave the monitor with `Ctrl+]`.
- All pins and timers are in `idf.py menuconfig` → **SolLimit** (Part 10.1).

A good first boot looks like this (some lines shortened):

```
W config: no saved config, using Kconfig defaults
I log: 496 sectors, 50592 records (14 h at 30 inverters / 30 s), next seq 0, oldest 0
I modbus: RS-485 on TX27 RX26 DE25, 9600 8N1
I curtail: 30 inverters, telemetry every 30 s, dry run
I modem: AIR780E on UART2 TX17 RX16 115200 baud, reset GPIO23, 4G uplink (PPP)
I hes: not activated yet: registration runs when the uplink is up
I main: AP "SolLimit-3C4D5E" up at 192.168.4.1
W portal: portal OPEN: join the SolLimit-xxxxxx Wi-Fi, go to http://192.168.4.1/ within 300 s
I modem: SIM ICCID 8994..., MSISDN (not on SIM)
I modem: module ready, CSQ 20 (-73 dBm)
```

ModbusPal starts flashing within 30 s, because the ESP32 polls all 30 inverters (still in **dry run**: it reads but never writes).

---

## Part 6. Commissioning from your phone

Pick an uplink:

- **Path A, Wi-Fi uplink (start here).** The ESP32 joins the same 2.4 GHz Wi-Fi as your PC. Everything stays on your LAN. The 4G module is still used for ICCID and signal strength.
- **Path B, 4G uplink (full field setup).** The ESP32 reaches the simulator over the cellular network. Your PC must then be reachable from the internet (Part 9). Do Path A first.

Steps:

1. Within 5 minutes of power-up (or after holding **Button 1** for 10 s), join the Wi-Fi **`SolLimit-XXXXXX`** from your phone. Password `sollimit123`.
2. When the phone warns "no internet", choose **Stay connected**.
3. Open `http://192.168.4.1/` and sign in with `sollimit-admin`.
4. Fill in the form:

| Field | Path A value | Path B value |
|---|---|---|
| Device ID | `SL-DEV-0001` | same |
| Customer account | `CUST-0001` | same |
| Inverters | `1-30` | same |
| Serial prefix | `INV-` (gives INV-001 ... INV-030) | same |
| RS-485 baud | `9600` | same |
| **Really write setpoints** | **tick it** (ModbusPal is not a real plant) | same |
| Telemetry interval | `30` (or 15) | same |
| Restore defaults after HES lost for (s) | `120` for the tests (use 1800 in the field) | same |
| NSO registration URL | `https://192.168.1.4:8443/api/v1/devices/register` | your public address (Part 9) |
| NSO MQTT broker | `mqtts://192.168.1.4:8883` | your public address (Part 9) |
| SIM APN | your SIM provider's APN | **required** |
| Site Wi-Fi SSID / password | your 2.4 GHz Wi-Fi | **leave empty** (empty = 4G) |
| Device AP password, API key | change them if you like | same |

You can also write inverters with their own serials: `1:SN12345,2:SN12346,3-30`.

5. Press **Save and reboot**. Then, on its own:

```
W main: UPLINK UP (Wi-Fi) 192.168.1.37
I main: data server: http://192.168.1.37:8080/
W main: clock set by NTP (pool.ntp.org)
I hes: registering SL-DEV-0001 at https://192.168.1.4:8443/api/v1/devices/register
W hes: ACTIVATED: prefix nso/v1/SL-DEV-0001; commissioning portal is now disabled
W portal: portal CLOSED (hold Button 1 for 10 s to ask NSO to open it)
I hes: MQTT starting: mqtts://192.168.1.4:8883 as SL-DEV-0001, prefix nso/v1/SL-DEV-0001
W hes: MQTT CONNECTED (v5, mTLS) to mqtts://192.168.1.4:8883, subscribed nso/v1/SL-DEV-0001/cmd/#
```

Window B shows the registration:

```
=== REGISTERED SL-DEV-0001 from 192.168.1.37
    firmware ...  hardware A
    SIM ICCID 8994...  MSISDN -
    customer CUST-0001  uplink wifi  30 inverters: INV-001, INV-002, ...
    topic prefix nso/v1/SL-DEV-0001  cert serial ...
```

Window C now shows the retained online status, then 30 telemetry messages every 30 s:

```
nso/v1/SL-DEV-0001/status {"state":"online","ts_ms":...,"inverters":30,"iccid":"8994...","rssi_dbm":-73,"mode":"remote"}
nso/v1/SL-DEV-0001/telemetry/INV-001 {"ts_ms":1790841600000,"seq":12,"device_id":"SL-DEV-0001","inv_sn":"INV-001","inv_addr":1,"p_kw":21.00,"q_kvar":0.10,"e_day_kwh":41.234,"v":[230.1,229.6,230.4],"i":[30.43,30.43,30.43],"sp_applied_pc":100,"q_applied_pc":0,"alarms":0,"comm_status":0,"rssi_dbm":-73,"backfill":false}
```

The first messages may have `"backfill":true`. Those are the records logged before activation, re-sent now.

**Data server (live table in a browser).** In Path A, open `http://<ESP32 IP>:8080/?key=change-me-key` from your PC. It shows the uplink, the HES state, the log, and a row for every inverter.

---

## Part 7. Topics and payloads

`{prefix}` = `nso/v1/SL-DEV-0001` (given by the registration).

| Topic | Direction | Retain | Payload |
|---|---|---|---|
| `{prefix}/cmd/setpoint` | HES → device | no | `{"task_id":"T0001","active_power":50,"reactive_power":0,"expiration":5,"inverters":["INV-005"]}` |
| `{prefix}/cmd/ack` | device → HES | no | `{"task_id":"T0001","execution_status":"success","active_power":50,"reactive_power":0,"expiration":"2026-10-01T10:05:00Z","targets":30,"applied":30,"failed":[],"actual_power_before":1065.00,"actual_power_after":1065.00,"execution_timestamp":"2026-10-01T10:00:02Z"}` |
| `{prefix}/telemetry/{sn}` | device → HES | no | see Part 6 (one per inverter, every 15–30 s) |
| `{prefix}/status` | device → HES | **yes** | online: as above. Last Will: `{"state":"offline"}` |
| `{prefix}/portal/request` | device → HES | no | `{"request_id":"9f2c...","device_id":"SL-DEV-0001","ts_ms":...,"wait_s":600}` |
| `{prefix}/cmd/portal` | HES → device | no | `{"request_id":"9f2c...","approved":true}` |
| `{prefix}/portal/status` | device → HES | no | `{"request_id":"9f2c...","state":"open"}` |

Command rules:

- `task_id`: 1–32 letters, digits, `-` or `_`. Each one is accepted **once, ever** (the last 64 are stored in flash).
- `active_power`: 0–100 (%). `reactive_power`: -100 to 100 (%). Send one or both.
- `expiration`: minutes from now (1–1440), **or** an absolute UTC time such as `"2026-10-01T12:30:00Z"`. When it passes, the inverters go back to 100 % / 0 %. Without `expiration` the setpoint stays until the next command.
- `inverters`: optional list of serials. Without it, the command goes to all inverters.
- `execution_status` in the ack: `success`, `partial` (some inverters failed, listed in `failed`), `failed`, `rejected` (with `reason`), or `expired` (sent when the expiry restores the defaults).
- `comm_status` in telemetry: 0 ok, 1 no reply, 2 bad reply. Values stay at the last good reading while it is not 0.
- `dry_run: true` in an ack means *Really write setpoints* is off: nothing was written.

---

## Part 8. Test plan: send commands, watch ModbusPal change

Open a fourth PowerShell window for sending:

```powershell
cd D:\Dinuk\itZorix\NSO\POC\SolLimit
$P = "D:\Dinuk\itZorix\NSO\POC\SolLimit\tools\hes\pki"
$TLS = @("-h","192.168.1.4","-p","8883","--cafile","$P\ca.crt","--cert","$P\operator.crt","--key","$P\operator.key","-V","mqttv5")
$T = "nso/v1/SL-DEV-0001"
function Send($file, $sub = "cmd/setpoint") { & "C:\Program Files\mosquitto\mosquitto_pub.exe" @TLS -q 1 -t "$T/$sub" -f $file }
```

(The payloads are sent from files with `-f`. Typing JSON straight into PowerShell 5.1 loses the quotes.)

Each `task_id` works only once. To repeat a test, open the JSON file in Notepad and change the `task_id` (for example `T0001` → `T0101`).

| # | Do this | Expect on the device / in Window C | Expect in ModbusPal |
|---|---|---|---|
| 1 | `Send tools\hes\commands\01_all_50pc_5min.json` | ack `success`, `targets 30, applied 30`. Telemetry `sp_applied_pc 50` | register 0 = **50** on all 30 slaves |
| 2 | Wait 5 minutes | log `command T0001 expired`, ack `expired` | register 0 back to **100** |
| 3 | `Send tools\hes\commands\02_inv005_30pc_3min.json` | ack `targets 1` | only **INV-005** register 0 = 30 |
| 4 | `Send tools\hes\commands\03_inv007_008_q-20pc.json` | telemetry `q_applied_pc -20` for INV-007/008 | register 1 = **65516** (that is -20) on slaves 7 and 8 |
| 5 | Send file 01 **again** (same `task_id`) | ack `rejected`, reason `task_id already processed (replay)` | nothing changes |
| 6 | Press the ESP32's **EN** (reset) button, wait for `MQTT CONNECTED`, send file 01 again | still `rejected`: replay protection survives a reboot | nothing |
| 7 | `Send tools\hes\commands\05_bad_range.json` | `rejected`, `active_power must be 0-100` | nothing |
| 8 | `Send tools\hes\commands\06_bad_unknown_inverter.json` | `rejected`, `unknown inverter serial` | nothing |
| 9 | Edit file 07 to a UTC time 3 minutes ahead (`(Get-Date).ToUniversalTime().AddMinutes(3).ToString("s") + "Z"` prints one), then send it | ack with that `expiration`; restored at that time | 70, then 100 |
| 10 | In ModbusPal **disable** slave 10, wait 30 s, then send a new command | telemetry INV-010 `comm_status 1`; ack `partial`, `failed:["INV-010"]` | others change, 10 does not |
| 11 | In ModbusPal type `1234` into INV-003 register 2 | next telemetry INV-003 `p_kw 12.34` | |
| 12 | Send a command (new task_id, 50 %), then stop the broker (Ctrl+C in Window A) | after 120 s: `FAIL-SAFE (HES lost): restoring 100 % / 0 %` | register 0 back to 100 |
| 13 | Start the broker again, wait about 1 minute | `MQTT CONNECTED`, then messages with `"backfill":true` and the `seq` numbers from the outage, then `backfill done` | |
| 14 | Unplug the ESP32's USB (power) | after about 90 s Window C shows `status {"state":"offline"}` (the Last Will) | |
| 15 | Portal re-open, see Part 8.1 | | |

If `mosquitto_sub` in Window C stopped when you stopped the broker, start it again.

Tests 12–13 use the 120 s HES-lost time from the commissioning form. In the field use 1800 s or more.

### 8.1 Re-open the portal after activation (FR-3.3, FR-3.4)

1. Hold **Button 1** for 10 s. Device log: `portal requested, waiting 600 s for NSO approval of request_id 9f2c4a...`.
2. Window C shows `nso/v1/SL-DEV-0001/portal/request {"request_id":"9f2c4a...",...}`.
3. Copy that `request_id` into `tools\hes\commands\portal_approve.json`, then:
   ```powershell
   Send tools\hes\commands\portal_approve.json "cmd/portal"
   ```
4. Device log: `NSO approved request 9f2c4a...: opening the portal`. The `SolLimit-XXXXXX` Wi-Fi appears, and Part 6 steps 1–3 work again for 5 minutes.

Without the approval, or with a wrong or old `request_id`, nothing opens. The button alone is never enough after activation.

**Recovery.** If the device can no longer reach NSO (wrong broker, APN, ...), nobody can approve. Then reflash it from a PC: `idf.py -p COM5 -b 115200 erase-flash flash`. That wipes the activation, so it registers again.

---

## Part 9. Path B: over 4G

The ESP32 now reaches your PC through the cellular network, so ports 8443 and 8883 on your PC must be reachable from the internet. Pick one option:

- **Router port forwarding.** Forward TCP 8443 and 8883 to your PC, and use your public IP or a dynamic-DNS name. This does not work if your ISP puts you behind carrier-grade NAT.
- **A free TCP tunnel**, for example playit.gg. Make one TCP tunnel to local port 8443 and one to 8883. Each gives you an address like `something.gl.at.ply.gg:12345`.
- **A small cloud VM.** Copy the `tools` folder to it and run Part 3 there.

Then:

1. Make the server certificate valid for the public name(s), and rebuild:
   ```powershell
   py tools\hes_sim.py init --force --host 192.168.1.4 --host something.gl.at.ply.gg --host other.gl.at.ply.gg
   idf.py build
   idf.py -p COM5 -b 115200 erase-flash flash monitor
   ```
2. Restart Windows A and B (the broker and the API read the new certificate).
3. Commission again (Part 6) with the **Site Wi-Fi SSID empty**, the right **APN**, and the public addresses, for example `https://something.gl.at.ply.gg:12345/api/v1/devices/register` and `mqtts://other.gl.at.ply.gg:23456`.

Expected log on 4G:

```
I modem: module ready, CSQ 20 (-73 dBm)
W main: UPLINK UP (4G) 10.64.12.7
W main: clock set by NTP (pool.ntp.org)
...
W hes: MQTT CONNECTED (v5, mTLS) ...
```

While the portal AP is open, the phone also gets internet through the 4G link (NAPT).

If the module refuses CMUX, the log says `CMUX refused, using plain PPP`. It still works, but `rssi_dbm` then stays at the value read before going online.

---

## Part 10. All configuration

### 10.1 menuconfig → SolLimit (compiled in: `idf.py menuconfig`, rebuild, flash)

| Setting | Default | Meaning |
|---|---|---|
| Device ID (default) | `SL-DEV-0001` | First-boot device ID / modem serial |
| Hardware revision | `A` | Sent at registration |
| **4G** TX / RX GPIO | 17 / 16 | |
| 4G baud | 115200 | |
| 4G RESET GPIO | 23 | -1 = not wired |
| Use CMUX | yes | Live signal strength while online. Falls back to plain PPP |
| **RS-485** TX / RX / DE GPIO | 27 / 26 / 25 | MAX485 DI / RO / DE+RE |
| **Portal** button GPIO | 18 | Button 1 |
| Hold time | 10 s | |
| Approval wait | 600 s | How long an NSO approval is accepted after the button |
| Login window | 300 s | |
| Session length | 900 s | |
| Portal password (default) | `sollimit-admin` | Change it in the portal |
| AP password (default) | `sollimit123` | |
| Wrong passwords before lock-out | 5 | then 60 s lock |
| NAPT for the AP | yes | Phones on the AP get internet |
| NTP server | `pool.ntp.org` | Re-sync every hour (`CONFIG_LWIP_SNTP_UPDATE_DELAY`), at boot and on every reconnect |
| Data server port / key | 8080 / `change-me-key` | |

### 10.2 Portal settings (saved in NVS)

The fields are listed in Part 6. When you save:

- Empty password fields keep their old value.
- Text cannot contain `"`, `\` or control characters. Device ID and serials allow only letters, digits, `-` and `_`.
- The device ID cannot change after activation, because it is in the certificate.
- The device reboots after saving.

### 10.3 What is stored where

| Partition | Size | Holds |
|---|---|---|
| `nvs` | 24 KB | settings, private key, certificate, topic prefix, used task_ids, running commands, backfill position |
| `factory` | 1.94 MB | the firmware (1.04 MB today, 47 % free) |
| `log` | 1.94 MB | the circular data log, 50,592 records |
| `coredump` | 64 KB | crash dump |

The log record is 40 bytes. It holds a UTC ms timestamp, seq, P (0.01 kW), Q (0.01 kVAr), daily energy (Wh), 3 voltages (0.1 V), 3 currents (0.01 A), alarms, inverter address, active and reactive setpoint applied, comm status, and the 4G RSSI in dBm. The device ID is the same for every record, so it is not stored per record. The serial is looked up from the address.

---

## Part 11. How it works inside

```
 ModbusPal (30 slaves) --USB-RS485== A/B ==MAX485-- UART1 (27/26/25)
                                                      |
                                       control task (curtail.cpp, only RS-485 user)
                                       | poll 30 inverters every 15-30 s
                                       | log every 30 s -> datalog.cpp (flash ring)
                                       | commands / expiry / fail-safe -> FC06 + read-back
                                       | publish telemetry, backfill
                                       v
 hes.cpp: registration (HTTPS + CSR) -> MQTT v5 / TLS 1.3 / mTLS client (esp-mqtt)
                                       |
                 uplink: site Wi-Fi (STA)  or  AIR780E PPP over UART2 (modem.cpp, esp_modem)
                                       |
                        HES simulator: hes_sim.py :8443 + Mosquitto :8883
```

| Task | File | Job |
|---|---|---|
| `control` | curtail.cpp | Only code that touches RS-485. Poll, log, publish, commands, expiry, fail-safe, backfill |
| `modem` | modem.cpp | Only code that touches UART2. Reset, ICCID/MSISDN/CSQ, PPP dial and redial |
| `hes` | hes.cpp | Waits for uplink + NTP, registers once, starts MQTT |
| `mqtt_task` | esp-mqtt | Connects, reconnects, delivers commands to curtail.cpp |
| `portal` | portal.cpp | Button, portal window, open/close |
| httpd x2 | portal.cpp, dataserver.cpp | The two web servers |

At start-up the firmware checks itself: the Modbus CRC, the inverter-list parser, the log ring arithmetic and the form decoder. If a code change breaks one, the board stops at boot with the failing line in the log.

---

## Part 12. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| ModbusPal COM list empty | Start it with the `java "-Djava.library.path=..."` command from Part 4.1 |
| ModbusPal never flashes, `modbus: addr 1 fc3: no reply` | Wrong COM port in ModbusPal (pick the converter's, not the ESP32's), ModbusPal not **Run**, A/B swapped, parity not None, MAX485 not powered, GPIO26/27 swapped |
| `bad reply` | Baud mismatch, or two masters on the bus |
| `read-back mismatch` | Slave not in ModbusPal, or register 0/1 missing |
| `module not answering AT` | TX/RX not crossed, no common GND, module not powered (5–24 V / 2 A), VCC pin not at 3.3 V |
| `dial failed (SIM? APN?)` / `no IP within 60 s` | APN wrong, SIM has no data, no antenna, no coverage. Path A does not need any of this |
| `not commissioned` | Registration URL / MQTT broker empty in the portal |
| `cannot reach https://...` | PC IP wrong, Window B not running, firewall (Part 3.2), ESP32 on another network |
| `registration refused: HTTP 400` | Window B prints the reason |
| `client_cert is not a valid certificate for this device's key` | The simulator was re-initialised mid-way. Retry; it registers again |
| `transport error: esp-tls ...` | Wrong host/port, broker not running, server certificate not valid for that host (re-run `init --host`), or the CA in the firmware is old (rebuild after `init`) |
| Mosquitto log `OpenSSL Error ... certificate` | The client's certificate is not from this CA. Erase and re-register the device |
| Connected, but no commands arrive | Wrong topic. It must be `nso/v1/<device id>/cmd/setpoint` |
| Device never reaches `clock set by NTP` | NTP blocked on your network. Set another NTP server in menuconfig. TLS needs the clock |
| Ack says `"dry_run":true` | Tick *Really write setpoints* (portal) |
| Portal page does not open | Not activated: power-cycle or hold Button 1 for 10 s. Activated: needs NSO approval (Part 8.1). A request from the site LAN gets 403 |
| `Wrong password` after a firmware change | Settings were reset to defaults: use `sollimit-admin` |
| Flash fails | Use `-b 115200`. Hold BOOT while it says `Connecting...` |
| `factory MAC blank` | Normal on this board. A fixed MAC is used |
| Board resets with `assert failed` | A boot self-check failed after a code change. The log line shows which |

---

## Part 13. Before production (not done in this POC)

| Item | Why | How |
|---|---|---|
| 7 days of log for 30 inverters | Internal flash holds about 14 h | Add a W25Q256 (32 MB) on SPI. Register it with `spi_bus_initialize` + `spi_bus_add_flash_device` + `esp_partition_register_external(..., "log")`. `datalog.cpp` already finds the partition by name |
| Protected key storage (FR-2.4) | NVS is plain text, so anyone holding the board can read the private key | Turn on flash encryption (`CONFIG_SECURE_FLASH_ENC_ENABLED`) and NVS encryption (`CONFIG_NVS_ENCRYPTION`, plus an `nvs_keys` partition). This burns eFuses and is permanent, so do it on production boards only |
| Real inverter register map | The map matches the simulator | Edit `R_*` and `poll_one()` in `curtail.cpp` |
| NSO's real PKI, API and broker | The simulator stands in for them | Put NSO's CA in `main/certs/hes_ca.pem`. Set the URL and broker in the portal |
| HTTPS portal | The portal is plain HTTP (the AP link itself is WPA2/WPA3) | `esp_https_server` |
| OTA updates | Dropped to make room for the log | Needs a second app slot, which needs the external flash first |

### Open points to confirm with NSO ([CHECK] items in the spec)

- The default restored on expiry (100 % active, 0 % reactive, in `kDefAp`/`kDefRp`).
- The `expiration` format (minutes, or an absolute UTC time: both are accepted now).
- `reactive_power` units (% of rated here).
- Topic names, payload format, alarm bits and comm status codes.
- How portal approval should reach the device (MQTT here).
- How many task_ids to keep (64 here).
- Whether syncing NTP every hour is fine (FR-6.2 asks for at least once a day).
- The storage figures: 350,000 records in 4 MB does not match 30 inverters × 7 days (about 605,000 records).

---

## Part 14. File list

```
SolLimit/
  partitions.csv            nvs, phy, factory app 1.94 MB, log 1.94 MB, coredump
  sdkconfig.defaults        PPP, NAPT, MQTT v5, TLS 1.3, CSR writing, cert dates, NTP 1 h
  main/
    app.hpp                 shared types: Config, LogRec, Inv, State, function list
    main.cpp                boot, Wi-Fi STA/AP, NAPT, NTP, network events
    config.cpp              settings, NVS, password hash, inverter-list parser
    modbus.cpp              Modbus RTU master (FC03/FC06, CRC16), any slave address
    curtail.cpp             poll/log/publish/commands/expiry/fail-safe/backfill (register map here)
    datalog.cpp             circular flash log
    hes.cpp                 registration (key + CSR), MQTT v5 mTLS, portal approval
    modem.cpp               AIR780E through esp_modem: reset, ICCID, CSQ, PPP/CMUX
    portal.cpp              commissioning portal
    dataserver.cpp          :8080 live view
    web.cpp                 form decoding, HTML escaping, AP-only check
    certs/hes_ca.pem        CA built into the firmware (made by hes_sim.py init)
  tools/
    hes_sim.py              HES simulator: init (PKI + broker config), serve (registration API), devices
    hes/mosquitto.conf      broker config (made by init)
    hes/acl.conf            topic permissions (made by init)
    hes/commands/*.json     test command payloads
    hes/pki/                CA and certificates (made by init; secret, git-ignored)
    modbuspal/SolLimit-30-inverters.xmpp   30 simulated inverters
```
