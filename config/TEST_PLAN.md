# CRUCIBLE UI Test Plan

Comprehensive manual test plan covering all frontend and backend functionality.
Uses `config/scenario.yaml` which includes 5 agents (UAV, UGV, USV, UUV),
all 4 sensor types, all 3 motion models, and 5 scenario events.

## Prerequisites

1. Build the workspace: `colcon build`
2. Source the workspace: `source install/setup.bash`
3. Launch the sim engine: `ros2 run sim_engine sim_engine`
4. Launch the WS bridge: `ros2 run ws_bridge ws_bridge`
5. Open the frontend in a browser

---

## 1. Initial Connection

| # | Action | Expected Result |
|---|--------|-----------------|
| 1.1 | Open the frontend URL | Map loads, status badge shows **PAUSED**, WS indicator shows **Connected**, agent list is empty, sim time shows `T: 0.000s` |
| 1.2 | Check browser console | No errors. `[WS] Bridge connected` logged. `get_state` sent automatically on connect |

---

## 2. Add Agents via Modal

| # | Action | Expected Result |
|---|--------|-----------------|
| 2.1 | Click **+ Agent** button | Modal opens with vehicle type dropdown, auto-generated ID (e.g. `uxv_01`), domain auto-filled to `1` |
| 2.2 | Change type to **UAV** | ID updates to `uav_01`, domain stays `1` |
| 2.3 | Enter lat/lon/alt/heading, click **Apply** | Modal closes, agent appears in sidebar list and on map, toast shows success |
| 2.4 | Click **+ Agent** again, leave type as UAV | ID should be `uav_02` (next available), domain `2` |
| 2.5 | Add a third UAV (`uav_03`) | ID increments correctly |
| 2.6 | Remove `uav_02` (via detail panel) | Agent disappears from list and map |
| 2.7 | Click **+ Agent**, type UAV | ID should recycle to `uav_02` (lowest available), domain `2` |
| 2.8 | Try adding agent with blank ID | Toast shows error "Agent ID is required" |

---

## 3. Place Agent on Map

| # | Action | Expected Result |
|---|--------|-----------------|
| 3.1 | Click the **Place UxV** button | Banner appears: "Click map to place agent", cursor changes |
| 3.2 | Click on the map | Agent placed at click location, banner disappears, agent appears in list |
| 3.3 | Use the dropdown arrow to switch to **UAV** | Place button label and icon update to "Place UAV" |
| 3.4 | Click **Place UAV**, then press **Escape** | Place mode cancels, no agent added |
| 3.5 | Place agents of each type (UAV, USV, UGV, UUV) | Each gets correct icon on map and in sidebar list |

---

## 4. Agent Selection and Detail Panel

| # | Action | Expected Result |
|---|--------|-----------------|
| 4.1 | Click an agent card in the sidebar | Card highlights, detail panel opens showing type, domain, position, altitude |
| 4.2 | Click the agent's map marker | Same agent selected, detail panel updates |
| 4.3 | Click a different agent | Selection changes, detail panel updates to new agent |
| 4.4 | Verify detail panel shows vehicle type/class | Type label and icon match the agent |

---

## 5. Sensor Management

| # | Action | Expected Result |
|---|--------|-----------------|
| 5.1 | Select an agent, click **+ Add Sensor** in detail panel | Sensor type dropdown appears (navsatfix, imu, altimeter, twr_radio) |
| 5.2 | Select **navsatfix**, click Add | Sensor badge appears on the agent card, toast shows success |
| 5.3 | Add **imu**, **altimeter**, **twr_radio** to the same agent | All 4 sensor badges visible, summary bar updates sensor count |
| 5.4 | Click **Remove** on a sensor (e.g. altimeter) | Sensor badge disappears, toast confirms removal, sensor count decreases |
| 5.5 | Verify via `ros2 topic list` that the removed sensor's topic is gone | Topic no longer published |
| 5.6 | Re-add the removed sensor | Sensor reappears, topic resumes publishing |

---

## 6. Load Scenario

| # | Action | Expected Result |
|---|--------|-----------------|
| 6.1 | Click **Load** button in toolbar | Config modal opens with empty text area |
| 6.2 | Paste contents of `config/scenario.yaml`, click **Apply** | Modal closes, 5 agents appear (uav_01, uav_02, ugv_01, usv_01, uuv_01) |
| 6.3 | Verify agent icons | uav_01/uav_02 = UAV icon, ugv_01 = UGV, usv_01 = USV, uuv_01 = UUV |
| 6.4 | Verify agent positions on map | 5 markers in the DC area, roughly clustered |
| 6.5 | Click **Fit Agents** button | Map zooms to show all 5 agents |
| 6.6 | Select uav_01 | Detail panel shows: type UAV/quadrotor, domain 1, 4 sensors listed |
| 6.7 | Select ugv_01 | Detail panel shows: type UGV/tracked, domain 3, 3 sensors |
| 6.8 | Select usv_01 | Detail panel shows: type USV/catamaran, domain 4, 3 sensors |
| 6.9 | Select uuv_01 | Detail panel shows: type UUV/torpedo, domain 5, 3 sensors |
| 6.10 | Check status badge | Should show **PAUSED** (scenario loads paused) |
| 6.11 | Check sim time | Should show `T: 0.000s` |

---

## 7. Save Scenario

| # | Action | Expected Result |
|---|--------|-----------------|
| 7.1 | With agents loaded, click **Save** | Modal opens with YAML content |
| 7.2 | Verify YAML contains `sim_dt: 0.01` | Uses new parameter name, not `tick_rate_hz` |
| 7.3 | Verify all 5 agents present with correct vehicle_type/class | Round-trip preserves all fields |
| 7.4 | Verify sensor configs are present | Each agent's sensors listed with noise params |
| 7.5 | Copy YAML, remove all agents, load the copied YAML | Agents restored identically |

---

## 8. Sim Controls — Play / Pause / Step

| # | Action | Expected Result |
|---|--------|-----------------|
| 8.1 | After loading scenario, click **Play** | Status badge changes to **RUNNING**, sim time starts advancing |
| 8.2 | Observe uav_02 on the map | Marker moves along its waypoint loop (diamond pattern) |
| 8.3 | Observe ugv_01 | Marker moves along its triangular patrol route |
| 8.4 | Observe uuv_01 | Marker moves northeast along its one-way path |
| 8.5 | Observe uav_01 and usv_01 | Markers stay put (static and commanded_velocity with no input) |
| 8.6 | Click **Pause** | Status badge changes to **PAUSED**, all motion stops, sim time freezes |
| 8.7 | Note the sim time, click **Step** | Time advances by exactly `sim_dt` (0.01s). Status stays **PAUSED** |
| 8.8 | Click **Step** 10 more times | Time advances by 0.01s each click |
| 8.9 | Click **Play**, then click **Step** | Step should fail (toast: "Cannot step while running"). Must pause first |

---

## 9. Sim Controls — Speed

| # | Action | Expected Result |
|---|--------|-----------------|
| 9.1 | With sim paused, change speed dropdown to **2x**, click Play | Sim runs, time advances ~2x faster than wall-clock |
| 9.2 | Change to **0.5x** while running | Sim slows down, agents move at half speed |
| 9.3 | Change to **5x** | Agents move noticeably faster |
| 9.4 | Change to **10x** | Fast-forward. Waypoint agents should complete loops quickly |
| 9.5 | Change to **Max** (0x) | Sim runs as fast as CPU allows. Sim time advances rapidly |
| 9.6 | Change back to **1x** | Normal speed resumes |
| 9.7 | Verify via `ros2 topic hz /<agent_id>/gps/fix` | At 1x: ~5 Hz. At 2x: ~10 Hz. At 0.5x: ~2.5 Hz |

---

## 10. Sim Controls — Reset

| # | Action | Expected Result |
|---|--------|-----------------|
| 10.1 | Run sim at 1x for ~10 seconds, then Pause | Agents have moved from initial positions, sim time > 0 |
| 10.2 | Click **Reset** | Status badge shows **PAUSED**, sim time resets to `T: 0.000s` |
| 10.3 | Verify agent positions on map | All agents back at their initial positions |
| 10.4 | Click Play | Agents start their motion patterns from the beginning again |
| 10.5 | Run for 5s, reset, run again | Waypoint agents follow identical paths (deterministic seed) |

---

## 11. Map Dragging and Set Pose

| # | Action | Expected Result |
|---|--------|-----------------|
| 11.1 | With sim paused at time 0, drag an agent marker | Marker moves to new position. This is an initial condition change |
| 11.2 | Click Reset | Agent returns to the *dragged* position (updated initial pose) |
| 11.3 | Run sim for 5s, pause | Agents have moved |
| 11.4 | Drag a static agent (uav_01) to a new position | Marker moves. This is a mid-run reposition |
| 11.5 | Click Reset | Agent returns to its *original* initial pose (not the mid-run drag) |
| 11.6 | Click Play | Status goes to RUNNING |
| 11.7 | Try to drag an agent marker | Dragging should be disabled while running |
| 11.8 | Click Pause | Dragging re-enabled |

---

## 12. Scenario Events (requires running sim with loaded scenario)

| # | Action | Expected Result |
|---|--------|-----------------|
| 12.1 | Load scenario, set speed to 5x, click Play | Sim runs at 5x speed |
| 12.2 | Wait until T > 30s | Event fires: `disable_sensor` on ugv_01/navsatfix. Verify via `ros2 topic hz /ugv_01/gps/fix` — should stop publishing |
| 12.3 | Wait until T > 60s | Event fires: `update_param` on uav_01/twr_radio `max_range_m` to 150. TWR ranges from uav_01 should now only include agents within 150m |
| 12.4 | Wait until T > 90s | Event fires: `update_param` on uav_02/navsatfix `horizontal_std_m` to 10.0. GPS readings on uav_02 should become much noisier |
| 12.5 | Wait until T > 120s | Event fires: `set_pose` on usv_01 to new lat/lon. USV marker should jump on the map |
| 12.6 | Wait until T > 180s | Event fires: `update_param` on uuv_01/altimeter `std_m` to 2.0. Depth readings become noisier |

---

## 13. TWR Radio Inter-Agent Ranging

| # | Action | Expected Result |
|---|--------|-----------------|
| 13.1 | Load scenario, run sim | Multiple agents have twr_radio sensors |
| 13.2 | `ros2 topic echo /uav_01/twr/ranges` | Should show ranges to nearby agents (uav_02, ugv_01, usv_01, uuv_01) that are within 500m and also have twr_radio |
| 13.3 | As uav_02 flies its loop, ranges from uav_01 should change | Distance values increase/decrease as uav_02 moves |
| 13.4 | After T>60s (range reduced to 150m), some agents may drop out of uav_01's range list | Fewer entries in the RangeArray |

---

## 14. Ground Truth Publishing

| # | Action | Expected Result |
|---|--------|-----------------|
| 14.1 | Run sim, check `ros2 topic list` | Each agent has `/<agent_id>/sim/ground_truth` topic |
| 14.2 | `ros2 topic echo /uav_02/sim/ground_truth` | Shows lat/lon/alt changing as agent moves along waypoints |
| 14.3 | `ros2 topic hz /uav_01/sim/ground_truth` | Should be ~50 Hz at 1x speed |
| 14.4 | Compare ground truth vs navsatfix for same agent | Ground truth should be exact; navsatfix has noise added |

---

## 15. WebSocket Reconnect and State Sync

| # | Action | Expected Result |
|---|--------|-----------------|
| 15.1 | With agents added, kill and restart the WS bridge | Frontend should detect reconnect, WS indicator briefly shows Disconnected then Connected |
| 15.2 | Check agent list after reconnect | Agents still present (state persisted in sim engine, re-fetched via get_state) |
| 15.3 | Kill and restart the sim engine (backend restart) | On reconnect, frontend detects stale backend (version 0 < last known), pushes UI state |
| 15.4 | Verify toast message | Should show "Backend restart detected — restoring state from UI" |

---

## 16. Agent Counter Recycling

| # | Action | Expected Result |
|---|--------|-----------------|
| 16.1 | Add uxv_01, uxv_02, uxv_03 via Place button | Three agents with sequential IDs |
| 16.2 | Remove uxv_01 | Agent removed |
| 16.3 | Place another UxV | New agent gets ID `uxv_01` (recycled lowest available) |
| 16.4 | Remove uxv_02 | Agent removed |
| 16.5 | Place another UxV | Gets `uxv_02` |
| 16.6 | Remove all agents, place one | Gets `uxv_01` (counter fully resets) |

---

## 17. Edge Cases

| # | Action | Expected Result |
|---|--------|-----------------|
| 17.1 | Load scenario twice in a row | Old agents cleared, new set loaded cleanly. No duplicates |
| 17.2 | Load scenario, add an agent manually, then load again | Manual agent is removed, scenario agents restored |
| 17.3 | Load invalid YAML in config modal | Error toast, no crash |
| 17.4 | Remove a sensor that doesn't exist (race condition) | Graceful error, no crash |
| 17.5 | Rapidly click Play/Pause/Play/Pause | Status toggles cleanly, no stuck state |
| 17.6 | Change speed while paused, then play | New speed takes effect immediately |
| 17.7 | Reset with no agents loaded | No error, time resets to 0 |
| 17.8 | Step with no agents loaded | Time advances, no error |
| 17.9 | Place agent while sim is running | Agent added, appears on map (not draggable while running) |
| 17.10 | Remove agent while sim is running | Agent removed from list, map, and sim engine cleanly |
| 17.11 | Load scenario, save it, compare | Saved YAML should match loaded config (modulo key ordering and formatting) |
| 17.12 | Disconnect network briefly, reconnect | WS auto-reconnects after 3s, state re-syncs |
