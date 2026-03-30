# CRUCIBLE

**C**ollaborative **R**obotics **U**nified **C**onfiguration, **I**nstrumentation, **B**enchmarking, and **L**ifecycle **E**nvironment

A multi-agent SITL (Software-In-The-Loop) framework for testing and evaluating distributed robotics systems. CRUCIBLE provides simulated sensor outputs over ROS2, a web-based interface for placing and configuring agents on a map, stack orchestration to launch full autonomy stacks per agent, and real-time pose estimate evaluation against ground truth.

## Architecture

CRUCIBLE runs as a Docker Compose stack with five components:

- **sim_engine** — ROS2 Jazzy Python node. Runs the simulation loop, manages agents and sensors, publishes simulated sensor data and ground truth on namespaced topics.
- **zenoh_bridge** — Zenoh ROS2 DDS bridge. Routes simulated sensor topics from the sim engine's DDS domain to agent stacks via Zenoh, mirroring how real inter-vehicle comms work.
- **ws_bridge** — ROS2 node + WebSocket server. Translates between the web frontend and ROS2 services/topics on the sim engine.
- **stack_orchestrator** — Standalone Python service with Docker socket access. Launches and stops per-agent Docker Compose stacks from the browser.
- **frontend** — Vanilla HTML/JS/CSS served by nginx. Leaflet map, agent management, sensor configuration, accuracy overlay.

## Quick Start

```bash
# Create the shared Docker network (if it doesn't exist)
docker network create --subnet=172.20.0.0/16 swarm_net

# Clone with submodules
git clone --recurse-submodules <repo-url> crucible
cd crucible

# Build the ROS2 workspace
colcon build

# Launch CRUCIBLE
docker compose up --build

# Open the UI
open http://localhost:8080
```

## Repository Structure

CRUCIBLE is organized as a colcon workspace with submodules under `src/`:

```
crucible/
├── docker-compose.yml
├── config/
│   ├── scenario.yaml               # Example scenario (3 UAVs)
│   └── zenoh_bridge.json5          # Zenoh bridge config
├── stacks/
│   └── agent_stack.yml             # Template agent stack compose
├── data/
│   └── terrain/                    # SRTM DEM tiles (optional)
├── src/
│   ├── crucible_msgs/              # Submodule — ROS2 message/service definitions
│   │   ├── msg/
│   │   │   ├── GroundTruth.msg
│   │   │   ├── RangeStamped.msg
│   │   │   └── RangeArray.msg
│   │   ├── srv/
│   │   │   ├── AddAgent.srv
│   │   │   ├── RemoveAgent.srv
│   │   │   ├── ConfigureSensor.srv
│   │   │   ├── LoadScenario.srv
│   │   │   └── SaveScenario.srv
│   │   ├── CMakeLists.txt
│   │   └── package.xml
│   │
│   ├── crucible_engine/            # Submodule — sim engine, WS bridge, frontend
│   │   ├── sim_engine/             # ROS2 Python package
│   │   │   ├── sim_engine/
│   │   │   │   ├── node.py         # Main node and sim loop
│   │   │   │   ├── agent.py        # Agent state representation
│   │   │   │   ├── world_state.py  # Agent registry and spatial queries
│   │   │   │   ├── terrain.py      # DEM elevation lookups
│   │   │   │   ├── config_loader.py
│   │   │   │   ├── scenario_runner.py
│   │   │   │   ├── plugin_discovery.py
│   │   │   │   ├── sensors/        # Sensor model plugins
│   │   │   │   │   ├── __init__.py # SensorModel ABC + registry
│   │   │   │   │   ├── navsatfix.py
│   │   │   │   │   ├── imu.py
│   │   │   │   │   ├── altimeter.py
│   │   │   │   │   └── twr_radio.py
│   │   │   │   └── motion/         # Motion model plugins
│   │   │   │       ├── __init__.py # MotionModel ABC + registry
│   │   │   │       ├── static.py
│   │   │   │       ├── waypoint.py
│   │   │   │       └── commanded.py
│   │   │   ├── package.xml
│   │   │   └── setup.py
│   │   ├── ws_bridge/              # ROS2 Python package
│   │   │   ├── ws_bridge/
│   │   │   │   ├── node.py
│   │   │   │   └── protocol.py
│   │   │   ├── package.xml
│   │   │   └── setup.py
│   │   ├── frontend/               # Web UI (served by nginx)
│   │   │   ├── index.html
│   │   │   ├── css/style.css
│   │   │   ├── js/
│   │   │   │   ├── app.js
│   │   │   │   ├── ws.js
│   │   │   │   ├── map.js
│   │   │   │   ├── agents.js
│   │   │   │   ├── sensors.js
│   │   │   │   ├── accuracy.js
│   │   │   │   ├── scenario.js
│   │   │   │   ├── orchestrator.js
│   │   │   │   └── sim_control.js
│   │   │   └── nginx.conf
│   │   └── docker/
│   │       ├── Dockerfile.sim_engine
│   │       ├── Dockerfile.ws_bridge
│   │       └── Dockerfile.frontend
│   │
│   └── crucible_orch/              # Submodule — stack orchestrator
│       ├── orchestrator/
│       │   ├── compose_manager.py
│       │   └── server.py
│       ├── docker/
│       │   └── Dockerfile
│       ├── requirements.txt
│       └── COLCON_IGNORE
└── README.md
```

## Features

### Sensor Simulation
Built-in sensor models with configurable noise and update rates:
- **NavSatFix** (GPS) — horizontal/vertical Gaussian noise on WGS84 position
- **IMU** — accelerometer, gyroscope, and orientation with per-axis noise
- **Altimeter** — barometric altitude with optional AGL mode using terrain data
- **TWR Radio** — mesh radio two-way ranging with max range and per-measurement noise

### Plugin System
Add custom sensors via separate ROS2 packages:
1. Implement the `SensorModel` ABC from `sim_engine.sensors`
2. Register via `@register_sensor("your_sensor")` decorator
3. Declare an entry point in `setup.py` under `sim_engine.sensors`
4. Mount into the sim engine container

Same pattern for custom motion models via `sim_engine.motion`.

### Reproducible Scenarios
Define complete test scenarios in YAML:
- Agent positions, sensors, motion models
- Per-sensor seeded RNG for deterministic runs
- Timed events (disable sensors, change parameters, teleport agents)
- Stack launch configuration per agent

### Stack Orchestration
Launch full autonomy stacks per agent directly from the browser:
- Click "Launch Stack" on an agent to spin up its Docker Compose stack
- Environment variables (AGENT_NAME, AGENT_ID, ROS_DOMAIN_ID, etc.) are injected automatically
- Monitor stack status in real time
- Tear down individual stacks or all at once

### Pose Evaluation
Subscribe to pose estimate topics from your estimator and see accuracy in real time:
- Ground truth vs. estimate overlay on the map
- Horizontal and vertical error metrics per agent
- Color-coded error thresholds

## Network Topology

```
swarm_net (172.20.0.0/16)
│
├── sim_engine (172.20.0.100)
│   ├── zenoh_bridge (shared network namespace)
│   └── ws_bridge (shared network namespace)
│
├── stack_orchestrator (172.20.0.101)
│
├── agent uav_01 stack
│   ├── comms_gateway (on swarm_net)
│   ├── zenoh_bridge (shared)
│   ├── estimator (shared)
│   └── ...
│
├── agent uav_02 stack
│   └── ...
```

Agent stacks use `ROS_AUTOMATIC_DISCOVERY_RANGE=LOCALHOST` to keep DDS traffic contained within each stack. Zenoh bridges handle all cross-stack communication, matching the real deployment topology where inter-vehicle traffic routes over mesh radios.

The sim engine publishes on its own DDS domain with a co-located Zenoh bridge that peers with the agent Zenoh bridges over the shared Docker network. From each agent stack's perspective, simulated sensor data arrives over Zenoh identically to how real inter-agent data arrives in the field.

## Writing Custom Sensor Plugins

```python
from sim_engine.sensors import SensorModel, TopicConfig, QoSPreset, register_sensor
from your_sensor_msgs.msg import YourSensorData

@register_sensor("your_sensor")
class YourSensorModel(SensorModel):
    def configure(self, params):
        self._rate_hz = params.get("rate_hz", 10.0)
        if "seed" in params:
            self.set_seed(params["seed"])

    def get_topic_config(self):
        return TopicConfig(
            suffix="your_sensor/data",
            msg_type=YourSensorData,
            qos=QoSPreset.SENSOR_DATA,
        )

    def update(self, agent, world, dt):
        if not self.should_publish(dt):
            return None
        msg = YourSensorData()
        # ... populate from agent state + noise ...
        return msg
```

Package `setup.py`:
```python
entry_points={
    'sim_engine.sensors': [
        'your_sensor = your_sensor_sim.model:YourSensorModel',
    ],
}
```

## Submodule Repositories

| Repository | Description |
|---|---|
| `crucible_msgs` | ROS2 message and service definitions. Depended on by the engine, custom sensor plugins, and agent stacks. |
| `crucible_engine` | Simulation engine, WebSocket bridge, and web frontend. The core CRUCIBLE tool. |
| `crucible_orch` | Stack orchestrator. Generic Docker Compose lifecycle manager, no ROS2 dependency. |

## Building

### colcon (native ROS2 build)
```bash
cd crucible
colcon build
source install/setup.bash
```

`colcon` discovers `crucible_msgs`, `sim_engine`, and `ws_bridge` under `src/`. The `crucible_orch` directory contains a `COLCON_IGNORE` marker since it is not a ROS2 package.

### Docker
```bash
docker compose up --build
```

All Dockerfiles live in `docker/` subdirectories within their respective submodules and use the integration repo root as their build context, referencing source paths under `src/`.

## License

MIT
