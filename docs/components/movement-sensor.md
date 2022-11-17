---
title: "Movement Sensor Component"
linkTitle: "Movement Sensor"
weight: 70
type: "docs"
description: "Explanation of movement sensor configuration and usage in Viam."
# SME: Rand
---
The movement sensor component is an abstraction of a sensor that gives data on where a robot is and how fast it is moving.

We have chosen to abstract these types of sensors into one common API.
There are many different types of sensors that can provide data for some or all of the following methods: `Position`, `Orientation`, `LinearVelocity`, `AngularVelocity` and `CompassHeadings`.
A global positioning system (GPS) can provide position, linear velocity and compass headings.
An inertial measurement unit (IMU) can provide angular velocity and orientation.
We can further apply algorithms, such as a <a href="https://en.wikipedia.org/wiki/Kalman_filter" target="_blank">Kalman filter</a>[^kalman], to combine data from both a GPS and an IMU to output the full set of information of the movement sensor methods.

Currently (01 November 2022), the [RDK](../../appendix/glossary/#rdk_anchor) implements GPS, IMU, and visual odometry-based movement sensors.
We support two IMU models (manufactured by WitMotion and VectorNav) and two GPS models: <a href="https://en.wikipedia.org/wiki/NMEA_0183" target="_blank">NMEA-based</a>[^nmea] GPS modules and <a href="https://en.wikipedia.org/wiki/Networked_Transport_of_RTCM_via_Internet_Protocol" target="_blank">NTRIP-based</a>[^ntrip] <a href="https://en.wikipedia.org/wiki/Real-time_kinematic_positioning" target="_blank">RTK</a>[^rtk] GPS models.
The `cameramono` RDK model is experimental and uses a camera to output data on its position and orientation.

We specifically cover GPS and IMU units in this documentation.
Find the more [generic sensor component here](../../components/sensor/).
Find more information about encoders, another component type, [here](../../components/encoder/).

[^kalman]: Kalman filter: <a href="https://en.wikipedia.org/wiki/Kalman_filter" target="_blank">ht<span></span>tps://en.wikipedia.org/wiki/Kalman_filter</a>

[^nmea]: NMEA 0183: <a href="https://en.wikipedia.org/wiki/NMEA_0183" target="_blank">ht<span></span>tps://en.wikipedia.org/wiki/NMEA_0183</a>

[^ntrip]: Network Transport of RTCM via Internet Protocol (NTRIP): <a href="https://en.wikipedia.org/wiki/Networked_Transport_of_RTCM_via_Internet_Protocol" target="_blank">ht<span></span>tps://en.wikipedia.org/wiki/Networked_Transport_of_RTCM_via_Internet_Protocol</a>

[^rtk]: Real-time kinematic positioning: <a href="https://en.wikipedia.org/wiki/Real-time_kinematic_positioning" target="_blank">ht<span></span>tps://en.wikipedia.org/wiki/Real-time_kinematic_positioning</a>

{{% alert title="Note" color="note" %}}
Any movement sensor that uses I<sup>2</sup>C must be connected to a board that supports I<sup>2</sup>C and [has it enabled](../../getting-started/rpi-setup/#enabling-specific-communication-protocols-on-the-raspberry-pi).
{{% /alert %}}

## GPS
A global positioning system (GPS) is based on receiving signals from satellites in the earth’s orbit.
A GPS is useful for knowing where you are and how fast you’re going.
`Position`, `CompassHeading` and `LinearVelocity` data are provided by all GPS modules.
Fix and Correction data are available by using the sensor GetReadings method, which is available because GPSes wrap the sensor component.

{{% alert title="Info" color="tip" %}}
As with any Viam component configuration, configuring a movement sensor component requires a `name` of your choosing, a `type` ("movement_sensor"), and a `model` (see sections below for options).
Each GPS model must have either serial or I<sup>2</sup>C communication configured.
[Click here to jump to the communication config attribute information.](#connection-configuration)
Some models require other attributes, as detailed in their respective sections below.
{{% /alert %}}

We have integrated the following GPS modules into Viam’s RDK:

### GPS-NMEA
This GPS model uses communication standards set by the National Marine Electronics Association (NMEA).
The `gps-nmea` model can be connected via and send data through a serial connection to any device, or employ an I<sup>2</sup>C connection to a board:

#### GPS-NMEA over USB/Serial

```json
{
    "depends_on": ["board"],
    "model": "gps-nmea",
    "name": "UBLOX GPS",
    "type": "movement_sensor",
    "attributes": {
        "connection_type": "serial",
        "serial_attributes": {
            "serial_baud_rate": 115200,
            "serial_path": "/dev/serial/by-path/<device_ID>"
        }
    }
}
```
Note that the example `"serial_path"` filepath is specific to serial devices connected to linux systems.
#### GPS-NMEA over I<sup>2</sup>C

```json
{
    "depends_on": ["board"],
    "model": "gps-nmea",
    "name": "UBLOX GPS",
    "type": "movement_sensor",
    "attributes": {
        "board": "board",
        "connection_type": "I2C",
        "i2c_attributes": {
            "i2c_baud_rate": 115200,
            "i2c_addr": 111,
            "i2c_bus": "<name_of_bus_on_board>"
        }
    }
}
```

#### GPS-NMEA Attributes

Name | Type | Default Value | Description
---- | ---- | ------------- | -----
`board` | string | - | Required for NMEA over I<sup>2</sup>C; the board connected to the chip. Not required for serial communication.
`disable_nmea` | bool | false | Optional. If set to true, changes the NMEA message protocol to RTCM when using a chip as a base station.
`connection_type` | string | - | "I2C" or "serial", respectively

### GPS-RTK

_**(Experimental!)**_

This model uses real time kinematic positioning (RTK)[^rtk].
`gps-rtk`, a module with a chip (<a href="https://www.sparkfun.com/rtk" target="_blank">such as one of these from Sparkfun</a>[^chips]) capable of generating positional accuracy of 2cm.
The chip requires a correction source to get to the required positional accuracy.
Our `gps-rtk` model uses an over-the-internet correction source (NTRIP)[^ntrip] and sends the data over serial or I<sup>2</sup>C.

[^chips]: Sparkfun RTK Chips: <a href="https://www.sparkfun.com/rtk" target="_blank">ht<span></span>tps://www.sparkfun.com/rtk</a>

As shown in the examples below, the `gps-rtk` model requires a `name`, `type` ("movement_sensor"), and `model` ("gps-rtk").
In the `attributes` section, it requires connection configuration (see ["Connection Configuration,"](#connection-configuration) below), `correction_source` ("ntrip"), and an `ntrip_attributes` struct containing these:

Name | Type | Default Value | Description
---- | ---- | ------------- | ----------
**Required:** | | |
`ntrip_addr` | string | - | The URL of the NTRIP server from which you get correction data. Connects to a base station (maintained by a third party) for RTK corrections.
`ntrip_username` | string | - | Username for the NTRIP server
`ntrip_password` | string | - | Password for the NTRIP server
 | | |
**Optional:** | | |
`ntrip_connect_attempts` | int | 10 | no | How many times to attempt connection before timing out
`ntrip_baud` | int | defaults to `serial_baud_rate` | no | Only necessary if you want NTRIP baud rate to be different from serial baud rate.
---

#### GPS-RTK with NTRIP over USB/Serial

Example config:

```json
{
    "depends_on": ["board"],
    "model": "gps-rtk",
    "name": "UBLOX GPS",
    "type": "movement_sensor",
    "attributes": {
        "connection_type": "serial",
        "correction_source": "ntrip",
        "serial_attributes": {
            "serial_baud_rate": 115200,
            "serial_path": "/dev/serial/by-path/<device_ID>"
        },
        "ntrip_attributes": {
            "ntrip_addr": "<ntrip_address>",
            "ntrip_baud": 38400,
            "ntrip_password": "<password>",
            "ntrip_path": "",
            "ntrip_username": "<username>"
        }
    }
}
```

#### GPS-RTK with NTRIP over I<sup>2</sup>C

Example config:

```json
{
    "depends_on": ["board"],
    "model": "gps-nmea",
    "name": "UBLOX GPS",
    "type": "movement_sensor",
    "attributes": {
        "board": "board",
        "connection_type": "I2C",
        "correction_source": "ntrip",
        "i2c_attributes": {
            "i2c_baud_rate": 115200,
            "i2c_addr": 111,
            "I2c_bus": "<name_of_bus_on_board>",
        },
        "ntrip_attributes": {
 		    "ntrip_addr": "<ntrip_address>",
            "ntrip_baud": 38400,
            "ntrip_password": "<password>",
            "ntrip_username": "<username>"
        }
    }
}
```

### RTK-Station

_**(Experimental!)**_

The experimental `rtk-station` allows you to configure your own correction source.
This does not provide any movement sensor data on its own, but can be linked to an RTK-ready GPS module on a moving robot and send that robot correction data over your own network, radio, or Bluetooth in areas where internet connectivity is limited, or where an NTRIP server is unavailable.
We have implemented this in a way that does not rely on an internet connection to get correction data for a moving GPS.

For all of the following RTK-station configurations, `children` is the list of one or more other GPS components that can take RTCM corrections.

#### RTK-Station using NTRIP

```json
{
    "model": "rtk-station",
    "name": "my-rtk-station",
    "type": "movement_sensor",
    "children": [
        "gps1"
    ],
    "attributes": {
        "connection_type": "serial",
        "ntrip_attributes": {
            "ntrip_addr": "<NTRIP_address>",
            "ntrip_baud": 38400,
            "ntrip_password": "<password>",
            "ntrip_path": "",
            "ntrip_username": "<username>"
        },
        "correction_source": "ntrip"
    }
}
```

#### RTK-Station using I<sup>2</sup>C

```json
{
    "model": "rtk-station",
    "name": "my-rtk-station",
    "type": "movement_sensor",
    "children": [
        "gps1"
    ],
    "attributes": {
        "board": "board",
        "connection_type": "serial",
        "i2c_attributes": {
            "i2c_baud_rate": 115200,
            "i2c_addr": 111,
            "i2c_bus": "<name_of_bus_on_board>",
       },
        "correction_source": "I2C",
        "loc_accuracy": 10,
        "svin": "time",
        "time_accuracy": 60
    }
}
```

#### RTK-Station using Serial/USB

```json
{
    "children": [
        "gps1"
    ],
    "attributes": {
        "board": "board",
        "connection_type": "serial",
        "serial_attributes": {
        	"serial_baud_rate": 115200,
 		    "serial_path": "/dev/serial/by-path/<device_ID>"
        },
        "correction_source": "serial"
    }
}
```
### Connection Configuration

{{% alert="Note" color="note" %}}
Applies to all GPS models.
{{% /alert %}}

Use `connection_type`(string) to specify "serial" or "I2C" connection in the main `attributes` config. Then create a struct within `attributes` for either `serial_attributes` or `i2c_attributes`, respectively.

#### Serial Config Attributes

For a movement sensor communicating over serial, you'll need to include a `serial_attributes` field containing:

Name | Type | Default Value | Description
---- | ---- | ------------- | -----
`serial_path` | string | - | The name of the port through which the IMU communicates with the computer.
`serial_baud_rate` | int | 115200 | The rate at which data is sent to the IMU. Optional.
---

```json
{
    "name": "<my-movement-sensor-name>",
    "type": "<TYPE>",
    "model": "<MODEL>",
    "attributes": {
        "<whatever other attributes>": "<example>",
        "connection_type": "serial",
        "serial_attributes": {
        	"serial_baud_rate": 115200,
 		    "serial_path": "<PATH>"
        }
    }
}
```

#### I<sup>2</sup>C Config Attributes
For a movement sensor communicating over I<sup>2</sup>C, you'll need a `i2c_attributes` field containing:

Name | Type | Default Value | Description
---- | ---- | ------------- | -----
`i2c_bus` | string | - | The name of the port through which the IMU communicates with the computer.
`i2c_addr` | int | - |
`i2c_baud_rate` | int | 115200 | The rate at which data is sent to the IMU. Optional.
---

```json
{
    "name": "<my-movement-sensor-name>",
    "type": "<TYPE>",
    "model": "<MODEL>",
    "attributes": {
        "<whatever other attributes>": "<example>",
        "connection_type": "I2C",
        "i2c_attributes": {
            "i2c_addr": 111,
            "i2c_bus": "1"
        }
    }
}
```

## IMU

An inertial measurement unit (IMU) can provide `AngularVelocity`, `Orientation`, and `CompassHeading` methods out of the box (ordered from most common to least common). Acceleration and Magnetometer data are available by using the Sensor `GetReadings` method, which IMUs wrap.  We have included IMUs from two manufacturers in our RDK.

### IMU Configuration

An IMU will be configured with type `movement sensor`. Viam currently (01 November 2022) supports two IMU models, manufactured by WitMotion and VectorNav. They are configured with model `imu-wit` or `imu-vectornav`, respectively.

#### IMU-WIT

Example IMU-WIT config:

```json
{
    "name": "myIMU",
    "type": "movement_sensor",
    "model": "imu-wit",
    "attributes": {
        "serial_path": "<PATH>",
        "serial_baud_rate": 115200
    },
    "depends_on": []
}
```

#### IMU-VectorNav

Example IMU-VectorNav config:

```json
{
    "name": "myIMU",
    "type": "movement_sensor",
    "model": "imu-vectornav",
    "attributes": {
        "board": "local",
        "spi": "1",
        "spi_baud_rate": 3800,
        "polling_freq_hz": 80,
        "chip_select_pin": "36"
  },
  "depends_on": []
}
```
##### IMU-VectorNav Attributes

Name | Type | Default Value | Description
----- | ----- | ----- | -----
`board` | string | - | The name of the board to which the device is wired
`spi` | string | The name of the SPI bus over which the device communicates with the board. On a Raspberry Pi, people often use the bus named “1.”
`chip_select_pin` | string | - | The board pin (other than the SPI bus pins) connected to the IMU chip. Used to tell the chip whether the current SPI message is meant for it or for another device.
`spi_baud_rate` | int | 115200 | The rate at which data is sent to the IMU.
`polling_frequency_hz` | int |

## Cameramono

_**(Experimental!)**_

We have integrated an experimental package that uses a visual odometry algorithm with dead reckoning to track the Position, Orientation, LinearVelocity and AngularVelocity of the camera’s frame.
The `cameramono` model can use any single camera with this algorithm. 

In a Viam configuration file, a camera used as a movement sensor will require a [`camera` type component](../../components/camera/) and then a `movementsensor` type component that depends on the `camera` component, and a `motion_estimation_config` based on the camera properties.

```json
{
    "components": [
        {
            "attributes": {},
            "depends_on": [],
            "model": "pi",
            "name": "example-board",
            "type": "board"
        },
        {
            "name": "myCamera",
            "type": "camera",
            "model": "webcam",
            "attributes": {},
            "depends_on": []
        },
        {
            "name": "movementCamera",
            "type": "movementsensor",
            "model": "cameramono",
            "attributes": {
                "camera": "myCamera",
                "motion_config": "see_vision_documentation"
                },
            "depends_on": [
            "myCamera"
            ]
        }
    ]
}
```

## Software Implementation
[Python SDK Documentation](https://python.viam.dev/autoapi/viam/components/movement_sensor/index.html)

<br>