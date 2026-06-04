# drone_controller — the ROS 1 (Noetic) drone flight controller

`drone_controller` is the **ROS 1 flight-controller package** for
[AirPost](https://github.com/jsoone24/NC_AirPost), the autonomous drone parcel-delivery system. It
runs on the drone's companion computer (Jetson), bridges the cloud backend to the flight controller
over **MAVROS**, flies delivery missions, and operates the parcel winch.

> **Status — legacy ROS 1 path.** In the integrated AirPost_Drone system this package has been
> **superseded by a ROS 2 / uXRCE-DDS node** (`ros2_ws/src/airpost_drone`), which talks to PX4
> directly over DDS. This package is the original MAVROS implementation, kept as the Noetic path.

## What it does

```
backend ──MQTT (command/downlink/…)──►  drone_controller  ──MAVROS/MAVLink──►  Pixhawk
        ◄──MQTT (data/<id>, uplink)───      (this pkg)      ◄──MAVROS telemetry──
```

- Subscribes to its MQTT command topics and republishes live telemetry (position, velocity, battery,
  mission status) back to the backend as `data/<drone_id>` and `command/uplink/DevStatusAns/<drone_id>`.
- On a delivery request it pushes the waypoint mission to the Pixhawk, arms, takes off, and switches
  to AUTO so the vehicle flies the route.
- At the drop waypoint it pauses and drives the winch motor to lower and release the parcel.

## Scripts (`scripts/`)

| File | Role |
|---|---|
| **`MQTT.py`** | Main node (`rosrun drone_controller MQTT.py`). The MQTT bridge + telemetry: subscribes to MAVROS topics (`/mavros/global_position/global`, `/rel_alt`, `/raw/gps_vel`, `/battery`), publishes telemetry to the broker, and routes the `ActuatorReq` / `DevStatusReq` commands. |
| **`DroneController.py`** | Mission logic over MAVROS services/topics: clears and pushes the waypoint mission (`mavros/mission/push`), arms, takes off, sets flight mode, and tracks `mavros/mission/reached`. At the tag/drop waypoint it triggers the winch. |
| **`DCmotor.py`** | The parcel winch driver: `Jetson.GPIO` control of the DC motor (clockwise / counter-clockwise / stop) to lower and release the package. |

## Prerequisites

**Software**
- ROS 1 **Noetic** + **MAVROS**
- Python packages: `paho-mqtt`, `Jetson.GPIO`

**Hardware**
- Pixhawk 4 connected over USB (MAVLink → MAVROS)
- Winch DC motor wired to the Jetson GPIO

## Build & run

```bash
# clone into your catkin workspace, under catkin_ws/src

# make the entrypoint executable, then build
chmod +x scripts/MQTT.py
cd catkin_ws && catkin_make

# launch
source devel/setup.bash
rosrun drone_controller MQTT.py
```

`launch/all_drone_control.launch` brings the node up together with its dependencies.
