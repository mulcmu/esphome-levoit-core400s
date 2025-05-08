# esphome-levoit-core400s
Modifications to make a Levoit Core 400s air purifier to use esphome local control.  The stock PCB has an ESP32-SOLO-1C with headers broken out to program.  All of the user interface and control seems to be implemented on the U2 MCU.  There is another power PCB that supplies 5V from mains input and motor control via one connector and another connector that goes to the PM 2.5.  Very similar to the [Core300s](https://github.com/mulcmu/esphome-levoit-core300s) .

U2 Pins 13/14 are serial RX/TX connection to the ESP32 GPIO16/17 via level shifting mosfets.  TP28 and TP33 by the ESP32 are respective test points for the serial connection.

Similar to the [Core300s](https://github.com/mulcmu/esphome-levoit-core300s) project, an ESP32 devboard to capture both sides of the traffic on TP28 and TP33 with a little Arduino [project](https://github.com/mulcmu/ESP32_dual_serial_log).  115200 8n1 same as before.  Command packets are identical to the Core300s.  The Status packet was very similar, some of the bytes were reordered.

Current firmware is 1.1.01 for ESP and 1.1.07 for MCU.  As delivered FW was 1.0.08 for ESP and 1.1.06 for MCU.  

Updated to use external component instead of custom component.

https://github.com/acvigue/esphome-levoit-air-purifier provides a similar esphome project that is a bit more polished.

#### TODO:

- Investigate OTA of stock hardware without disassembly.
- Filter Life tracking
- Timer feedback if enabled on unit
- PM2.5 feedback seems very low.  Sensor unit is a Cubic PM2008MS.  Look into raw data.

#### Notes:

OTA firmware updated ESP and MCU firmware.  Both are seperate downloads.

Internal ESP32 seems to use MQTT to communicate with Vesync servers.  Wireshark dissector shows packets as malformed.  They might be encrypted.

The Core400s is much harder to disassemble to get to the UI pcb.  The bottom half unscrews similar to the Core300s.  The motor can be partially removed but not far enough to disconnect wires to UI pcb.  The top grate has snap fit connectors with a breach lock feature.  Once snapped down the whole vent is rotated to lock connections secure.  Once rotated into place, the snap connections prevent counter rotation to remove the vent.  A secure connection without any  screws but not very friendly for disassembly and repair without custom tooling.  

#### Packet Header:

`A5` start byte of packet

`22` send message or `12` ack message  (`52` might be error response)

`1-byte` sequence number, increments by one each packet

`1-byte` size of payload (after checksum byte)

`0` always zero

`1-byte` checksum.  Computed as 255 - ( (sum all of bytes except checksum) % 256 )

Payload Type:  3 bytes

The acknowledge packet `12` contains the same sequence number, with the `payload type` and `0`

#### Payload Types/Structure:

##### `01 B0 40` - Status (MCU to ESP)

The MCU sends a status packet on state changes or once every 60 seconds.

`01 B0 40 00 07 01 01 01 00 02 02 64 01 00 01 FF FF 00 00 00 00 02`

22 byte long (0x16) status packet payload:

Byte 4 Always `0`

Byte 5 MCU FW SubMinor

Byte 6 MCU FW Minor

Byte 7 MCU FW Major

Byte 8 Power:  

- `00` Off
- `01` On 

Byte 9 Fan mode:

- `00` Manual
- `01` Sleep
- `02` Auto 

Byte 10 Current Fan Speed

- `00` Sleep
- `01` Low
- `02` Med
- `03` High
- `04` Turbo
- `255` Power Off ??

Byte 11 Manual Fan Speed Selected

- `01` Low
- `02` Med 
- `03` High
- `04` Turbo

Byte 12 Screen Brightness

- `00` Screen Off
- `64` Screen Full (Screen illuminates briefly when another button is tapped while screen is off)

Byte 13 Screen

- `00` Off
- `01` On 

Byte 14 Always `0`

Byte 15 PM2.5  (Normally `1` increased to `4` during filter testing, color ring LEDs)

Byte 16 & 17 PM2.5  0xFFFF when off, samples every 15 minutes when off

Byte 18 Display Lock:

- `00` Unlocked
- `01` Locked

Byte 19 Fan Auto Mode: (Only configurable by the app)

- `00` Default, speed based on air quality
- `01` Quiet, air quality but no max speed
- `02` Efficient, based on room size

Byte 20 & 21 Efficient Area: 

- `BD 00` 100 sq ft (App Minimum)  0x00BD is 189

- `C1 07` 1050 sq ft (App Maximum) 0x07C1 is 1985

  Linear scale, not sure what the units are.  (sq ft)*(1.8905)-(0.5626) conversion

Byte 22 Always `00` or `02`

##### `01 E6 A5` - Configure Fan Auto Mode (ESP to MCU)

Byte 4 Always `0` 

Byte 5Fan Auto Mode

- `00` Default, speed based on air quality
- `01` Quiet, air quality but no max speed
- `02` Efficient, based on room size

Byte 6 & 7 `00 00` or Efficient Area

##### `01 60 A2` - Set Fan Manual (ESP to MCU)

Byte 4 Always `0`

Byte 5 Always `0`

Byte 6 Always `1`

Byte 7 Fan Speed:

- `01` Low
- `02` Med
- `03` High
- `04` Turbo

##### `01 E0 A5` - Set Fan Mode (ESP to MCU)

Byte 4 Always `0`

Byte 5 Fan Mode

- `00` Manual (App always uses `01 60 A2` with speed to change to manual mode)
- `01` Sleep
- `02` Auto 

##### `1 0 D1` - Display Lock (ESP to MCU)

Byte 4 Always `0`

Byte 5 Display Lock:

- `00` Unlocked
- `01` Locked

##### `01 29 A1` - Wifi LED state (ESP to MCU)

Wifi LED toggled at startup and when network connection changes

`1	29	A1	0	0	F4	1	F4	1	0 `Off

`1	29	A1	0	1	7D	0	7D	0	0` On

`1	29	A1  0   2   F4  1   F4  1   0` Blink  <!--TODO confirm on 400s-->

##### `01 B1 40` - Request Status (ESP to MCU)

Similar to status packet, occurs when Wifi led state is changed.  

##### `01 00 A0` - Set Power State (ESP to MCU)

Byte 4 Always `0`

Byte 5 Power State:

- `00` Off
- `01` On

##### `01 05 A1` Set Brightness (ESP to MCU)

Byte 4 Always `0`

Byte 5 Screen Brightness

- `00` Screen Off
- `64` Screen Full

##### `01 E2 A5` - Set Filter LED (ESP to MCU)

Byte 4

- `00` Off
- `01` On

Byte 5 `00`

Wasn't in original 300s captures before firmware update

##### `01 E4 A5` - Reset Filter (ESP to MCU and MCU to ESP)

Byte 4 `0`

Byte 5 `0`

MCU sends to ESP when sleep button held for 3 seconds

ESP sends to MCU when reset on app.

##### `01 65 A2` - Timer Status (MCU to ESP)

MCU sends a packet when timer is running with remaining time

`A5 12 27 0C 00 DA 01 65 A2 00 08 0D 00 00 10 0E 00 00`

Remaining time and initial time.

0x0D08 remaining seconds

0x0E10 initial seconds

##### `1 64 A2` Set Timer Time (ESP to MCU)

Byte 4 Always `0`

Byte 5 & 6 Time

Byte 7 & 8 Always `0`

##### `1 66 A2` Timer Start or Clear (MCU to ESP)

Byte 4 to 12 All `0` to Clear

Byte 5 & 6 and Byte 9 &10 set to same initial timer value

#### ESPHome functions:

- Toggle Power
- Set fan Mode, Manual, Sleep, Auto
- Set fan speed for manual mode
- PM2.5 sensor feedback
- Filter life remaining.  
- No Timer/Schedule support (implement in HA instead of on device)

#### Filter Life:

Manufacturer lists the filter life at 6 to 8 months and provided clean air deliver rates (CADR) for the different fan modes.

| Fan Speed | CADR    |
| --------- | ------- |
| Sleep     | 59 CFM  |
| Low       | 88 CFM  |
| Med       | 118 CFM |
| High      | 177 CFM |
| Turbo     | 260 CFM |

Filter lifetime air volume will be estimated based on user provided number of months and 24h operation on High.  The actual air volume through the filter will be estimated based actual runtime and volume for each fan speed.

Filter state sensor will provide a remaining lifetime percent.  Filter service interval will also be selectable and tracked.  Filter service is basically just to vacuum pre-filter.

