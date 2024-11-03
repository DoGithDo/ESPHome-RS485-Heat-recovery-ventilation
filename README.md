# ESPHome-RS485-Heat-recovery-ventilation
# Overview <!-- omit from toc -->
This project describes the smartification of a heat recovery ventilation system. ESPHome and RS485 are used between master and slave. In a first version, this project tried to use I2C, but failed (see https://github.com/DoGithDo/ESPHome-I2C-Master-Slave).

# Table of contents <!-- omit from toc -->
- [Heat recovery ventilation (HRV)](#heat-recovery-ventilation-hrv)
  - [Existing Setup](#existing-setup)
  - [New Setup](#new-setup)
    - [Concept](#concept)
    - [Master](#master)
    - [Slave](#slave)
    - [Mirroring](#mirroring)
    - [Self healing](#self-healing)
    - [Functions in Home Assistant](#functions-in-home-assistant)
- [Software](#software)
  - [Concept](#concept-1)
  - [Messages](#messages)
    - [Structure](#structure)
    - [Sending](#sending)
  - [Master](#master-1)
    - [Request Data](#request-data)
    - [Analyse Queue](#analyse-queue)
  - [Slave](#slave-1)
  - [Update of ESPHome entities using C++](#update-of-esphome-entities-using-c)
  - [RS485](#rs485)
    - [Master to Slave](#master-to-slave)
      - [Initialization](#initialization)
      - [Sending Data](#sending-data)
      - [Requesting Data](#requesting-data)
    - [Slave to Master](#slave-to-master)
      - [Initialization](#initialization-1)
      - [Sending Answers](#sending-answers)
      - [Receiving Commands](#receiving-commands)
- [Experiences](#experiences)
  - [Positive](#positive)
  - [How to improve](#how-to-improve)

# Heat recovery ventilation (HRV)
## Existing Setup
There is a ventilation system with a heat exchanger for the flat in my basement. The ventilation is controlled via a rotary switch in the flat. The ventilation system decides whether the flap of the heat exchanger is open or closed based on the measured temperature of the fresh air.
The ventilation system consists of the following components:
- A fan for the fresh air
- A fan for the exhaust air
- Two temperature sensors with the option of setting when the flap is opened or closed
- A motor that opens or closes the flap
- A rotary switch in the flat, with 4 positions (off, 1, 2, 3), which is connected to the basement via an approx. 10 m long cable

There is no WLAN in the basement.


## New Setup
### Concept
The ventilation should be made ‘smart’. The following components are used from the existing setup:
- The two fans
- The motor for the ventilation flap
- The rotary switch (with the existing cable to the new master)
- The existing cable from flat (master) to basement (slave)

As there is no WLAN reception in the basement, communication between the flat and the basement should take place via the existing cable.
The plan was to have an ESP32 (master) in the flat, which has a connection to the Internet / Home Assistant / CO2 sensors in the flat. In the basement, an ESP32 (slave) with various sensors should control the fans and the flap. The communication between the two ESPs uses RS485 (using a RS485 transceiver module ).

### Master
The master in the flat has the following functionality:
- Mirrors all sensors from the slave so that they can be read by the user or by Home Assistant.
- Receives commands from the Home Assistant (via WLAN) and sends them to the slave via RS485.
- If the master has no connection to Home Assistant, it behaves autonomously (e.g. deciding whether summer or winter)

### Slave
The slave receives the commands from the master and supplies the data from the sensors to the master via RS485.
The most important components of the slave:
- Control of the two fans
- Control of the flap (relay)
- Temperature sensor for fresh air and exhaust air
Other sensors are also available:
- Measured speed of the two fans
- Measurement of power consumption
- Additional temperature sensors, for difference fresh air / exhaust air before or after the heat exchanger

Additional functions:
- The slave also has an autonomous mode in case the RS485 communication fails.
- The slave can be set to access point mode via a command from the master (via RS485).

### Mirroring
All sensors of the slave are mirrored on the master. Example: The temperature of the fresh air is measured by the slave and then regularly sent to the master. It is available on the master in a template sensor.

### Self healing
If the connection between master and slave is interrupted, information is lost due to poor quality or a device restarts, the system heals itself.
Example: The master sends the command to the slave to close the flap. If the slave reports the status as open, the master sends the command to close again.

### Functions in Home Assistant
A few ideas on how Home Assistant automates the master, not described in detail:
- Stronger ventilation with more people in the home
- Increased ventilation in case of poor air quality (CO2, VOC)
- When the temperature is high in summer, the ventilation system should ventilate more strongly when the outside temperature is cooler (at night).
- If the tumble dryer in the flat heats up, the ventilation should run depending on the heating period.

# Software
## Concept
The ESP32 uses ESPHome. Unfortunately, ESPHome does not deliver any functions for a RS485 (or Modbus) client. These functions were therefore implemented in C++. This project already existed as a variant based on I2C. It was very easy to convert from I2C to RS485. This is why the terms master - slave are used.

Basic functionality:
- The entities (sensors, switches, ...) send messages between master and slave.
- These messages are constructed and parsed by a C++ package
- The messages are precalculated on the slave and then placed in a queue (C++ vector). From there they are picked up by the master.
- On the master, the received messages are also placed in a queue (C++ vector) and are processed later.

## Messages
### Structure
Examples of a message:

`TUB:22.6875#137$3*` (see explanation)

`PWF:-26.8848#248$1*`

`FHS:false#38$0*`

Explanation:
| Element   | Description                                                                                            |
| --------- | ------------------------------------------------------------------------------------------------------ |
| `TUB`     | Commands (type of message). These are defined as `enum` in handler-common.h                        |
| `:`       | Delimiter between command and payload.                                                                 |
| `22.6875` | Payload.  The payload is the e. g. the state of a sensor. In this case the temperature.                |
| `#`       | Delimiter between payload and checksum.                                                                |
| `137`     | Checksum                                                                                               |
| `$`       | Delimiter between cheksum and number of elements waiting in the queue. In case of master also end tag. |
| `3`       | Slave only: number of elements in the queue waiting to be delivered to the master                      |
| `*`       | Slave only: end tag                                                                                    |

### Sending
Master and Slave send messages.

Example in a switch:
```
    on_turn_on:
    - lambda: |-
        message("FHS", id(flap_heat_exchanger).state);
```

The `lambda` calls a C++ function. Parameter 1 is the command / type of message. Parameter 2 is the sensor value.

Example in an intervall:

```
- interval: ${update_interval_short}
    startup_delay: 3s
    then:
      - lambda: |-
          message("FFS", id(fan_fresh).speed);
```
Example on other triggers (here a fan):
```
    on_speed_set:
        - lambda: |-
            message("FFS", id(fan_fresh).speed);
```

## Master
The master periodically requests data from the slave and analyses the data in the queue. These functions are called in an interval.

```
interval:
 - interval: 0.1s
   then:
   - lambda: |-
      analyse_queue();
 - interval: 0.2s
   then:
   - lambda: |-
      request_data();
```
### Request Data
The master periodically requests the slave to supply data. The interval is defined as very short. However, the C++ function checks how many messages are still waiting in the slave's queue and adjusts the time until the next request accordingly.

### Analyse Queue
The data received from the slave is not analysed immediately, but written to an incoming queue. Function `analyse_queue` parses the message, checks the checksum and updates the ESPHome entities.

## Slave
On the slave, two queues are used:
1. The first one `response_queue` stores the precalculated messages. When the master requests data, the next message is delivered.
2. The second one `received_queue` stores the incoming commands. They are executed later.

## Update of ESPHome entities using C++
The ESPHome entities are directly updated using C++.

| Example                                                                      | Description                                                                        |
| ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `id(temp_fresh_before_mirror).publish_state(std::stof(response_value))`      | Updating a mirrored sensor on master. Converting string to float before publishing |
| `id(fan_fresh_mirror).make_call().set_speed(stoi(response_value)).perform()` | Updating the speed of a fan. Converting string to integer before updating          |
| `id(flap_heat_exchanger).turn_off()`                                         | Turning off a switch                                                               |

During programming, these direct calls were very unpleasant. The development environment used (VSCode) did not recognise the ESPHome entities in C++.
I have spent some time reading up on Intellisense but have not found a solution to this (cosmetic) problem.

## RS485
### Master to Slave
#### Initialization
The bus is initialized using C++, calling the function in YAML in `on_boot`.

```
Serial1.begin(BAUD, SERIAL_8N1, RX_PIN, TX_PIN)
```

#### Sending Data
Used code for sending the data:

```
Serial1.write(send_message.c_str());
```

#### Requesting Data
```
Serial1.write(REQUEST)
```
`REQUEST` contains a special command to trigger the slave to send data.

> [!NOTE]
>Im some cases I had timing issues: When I read the buffer, it was sometimes empty. The next time I read it, there were two responses in the buffer. The solution was to read the buffer until it was empty, but only read complete messages.

Read the buffer, as long as there is some data in the buffer:

```
while (Serial1.available())
    {
        read_buffer();
    }
```
Read only one message:

```
void read_buffer()
..
while (Serial1.available() && peek_delimiter != DELIMITER_QUEUE) 
    {
        peek_delimiter = Serial1.peek();
        response_message += (char)Serial1.read();
    }
```

### Slave to Master
#### Initialization
Same as for Master to Slave.

#### Sending Answers
I use a lambda in YAML to check the buffer in each loop:

```
  on_loop: 
    then:
      - lambda: |-
          receive_data();
```
If the received message was a `REQUEST`, the next waiting message in the queue is sent back.

#### Receiving Commands
The same functionality is used as in [Sending Answers](#sending-answers).
The command is written to the `received_queue` and executed later.

# Experiences
## Positive
- RS485 is way more reliable compared to I2C. The connection never got unstable.
- I wanted to do the project with ESPHome because you can implement many functions very easily (all sensors, web server, WiFi, ...). This has been confirmed.

## How to improve
- Installing WiFi in the basement would have provided a much simpler solution.
- This approach with queues is overengineered for RS485.  But it was easier to adapt the existing I2C solution.
- Building an own component in ESPHome or a custom component much easier? Way over my head.
