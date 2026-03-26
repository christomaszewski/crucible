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

# Launch CRUCIBLE
docker compose up --build

# Open the UI
open http://localhost:8080
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

### Scenario Configuration
Define complete test scenarios in YAML:
- Agent positions, sensors, motion models
- Per-sensor seeded RNG for reproducible runs
- Timed events (disable sensors, change parameters, teleport agents)
- Stack launch configuration per agent

### Stack Orchestration
Launch full autonomy stacks per agent directly from the browser:
- Click "Launch Stack" on an agent to spin up its Docker Compose stack
- Environment variables (AGENT_ID, ROS_DOMAIN_ID) are injected automatically
- Monitor stack status in real time
- Tear down individual stacks or all at once

### Pose Evaluation
Subscribe to pose estimate topics from your estimator and see accuracy in real time:
- Ground truth vs. estimate overlay on the map
- Horizontal and vertical error metrics per agent
- Color-coded error thresholds

## Project Structure

```
crucible/
├── docker-compose.yml          # Main CRUCIBLE stack
├── config/
│   ├── scenario.yaml           # Example scenario
│   └── zenoh_bridge.json5      # Zenoh bridge config
├── stacks/
│   └── agent_stack.yml         # Template agent stack compose
├── data/terrain/               # SRTM DEM tiles (optional)
├── sim_msgs/                   # ROS2 message/service definitions
├── sim_engine/                 # Simulation engine ROS2 package
│   └── sim_engine/
│       ├── node.py             # Main node and sim loop
│       ├── agent.py            # Agent state representation
│       ├── world_state.py      # Agent registry and spatial queries
│       ├── terrain.py          # DEM elevation lookups
│       ├── config_loader.py    # YAML config loading/saving
│       ├── scenario_runner.py  # Timed event execution
│       ├── plugin_discovery.py # Entry point plugin loading
│       ├── sensors/            # Sensor model plugins
│       └── motion/             # Motion model plugins
├── ws_bridge/                  # WebSocket bridge ROS2 package
├── frontend/                   # Web UI
│   ├── index.html
│   ├── css/style.css
│   └── js/
│       ├── app.js              # Main entry point
│       ├── ws.js               # WebSocket client
│       ├── map.js              # Leaflet map management
│       ├── agents.js           # Agent CRUD and UI
│       ├── sensors.js          # Sensor configuration
│       ├── accuracy.js         # Pose error overlay
│       ├── scenario.js         # Config load/save
│       ├── orchestrator.js     # Stack launch/stop
│       └── sim_control.js      # Play/pause/step/speed
├── crucible_orch/              # Stack lifecycle manager (submodule)
├── crucible_msgs/              # ROS2 message definitions (submodule)
```

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

## Network Topology

```
swarm_net (172.20.0.0/16)
│
├── sim_engine (172.20.0.100)
│   ├── zenoh_bridge (shared network)
│   └── ws_bridge (shared network)
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

Agent stacks use `ROS_AUTOMATIC_DISCOVERY_RANGE=LOCALHOST` to keep DDS traffic contained. Zenoh bridges handle all cross-stack communication, matching the real deployment topology.

## License

MIT
