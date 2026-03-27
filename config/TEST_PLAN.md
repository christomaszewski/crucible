# CRUCIBLE UI Test Plan

Comprehensive manual test plan covering all frontend and backend functionality.
Uses `config/scenario.yaml` which includes 5 agents (UAV, UGV, USV, UUV),
all 4 sensor types, all 3 motion models, and 5 scenario events.

## Prerequisites

1. Build the workspace: `colcon build`
2. Source the workspace: `source install/setup.bash`
3. Launch via docker compose or manually:
   - Sim engine: `ros2 run sim_engine sim_engine`
   - WS bridge: `ros2 run ws_bridge ws_bridge`
4. Open the frontend in a browser (use incognito or hard-refresh to avoid cache)

---

## 1. Initial Connection

| # | Action | Expected Result |
|---|--------|-----------------|
| 1.1 | Open the frontend URL | Map loads with dark tiles, status badge shows **PAUSED**, WS indicator shows **Connected** |
| 1.2 | Verify zoom controls | Zoom +/- buttons are in the **bottom-right** of the map (not top-left) |
| 1.3 | Verify filter bar | Vehicle type filter checkboxes visible in top-right of map with color swatches |
| 1.4 | Check browser console | No errors. `[WS] Bridge connected` logged. `get_state` sent automatically |

---

## 2. Add Agents via Modal

| # | Action | Expected Result |
|---|--------|-----------------|
| 2.1 | Click **+ Agent** button | Modal opens with vehicle type dropdown, auto-generated ID (e.g. `uxv_01`), domain auto-filled to `1` |
| 2.2 | Change type to **UAV** | ID updates to `uav_01`, domain stays `1` |
| 2.3 | Enter lat/lon/alt/heading, click **Add** | Modal closes, agent appears in sidebar list and on map, toast shows success |
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
| 3.3 | Click the dropdown arrow next to Place button | Dropdown shows all 5 vehicle types with representative icons |
| 3.4 | Switch to **UAV** in dropdown | Place button label and icon update to "Place UAV" |
| 3.5 | Click **Place UAV**, then press **Escape** | Place mode cancels, no agent added |
| 3.6 | Place agents of each type (UAV, USV, UGV, UUV) | Each gets correct shaped marker on map and icon in sidebar |

---

## 4. Map Markers

| # | Action | Expected Result |
|---|--------|-----------------|
| 4.1 | Verify marker shapes | UAV=diamond, UxV=circle, USV=pentagon, UGV=square, UUV=inverted triangle |
| 4.2 | Verify marker colors | Each type has a distinct color (cyan, blue, green, orange, purple) |
| 4.3 | Verify agent ID labels | Each marker shows the agent ID text below the shape |
| 4.4 | Click an agent marker on the map | Marker gets a glow effect, detail panel opens |
| 4.5 | Click the same marker again | Selection toggles off, detail panel closes |
| 4.6 | Load scenario, run sim, observe moving agents | Marker shapes rotate to match heading. Labels stay upright |

---

## 5. Vehicle Type Filter

| # | Action | Expected Result |
|---|--------|-----------------|
| 5.1 | With multiple agent types on map, uncheck **Air** filter | All UAV markers disappear from map |
| 5.2 | Re-check **Air** | UAV markers reappear |
| 5.3 | Uncheck **Ground** | UGV markers hidden |
| 5.4 | Uncheck multiple types simultaneously | Only checked types visible |
| 5.5 | Verify filter swatches | Color swatches dim when unchecked |

---

## 6. Agent Selection and Detail Panel

| # | Action | Expected Result |
|---|--------|-----------------|
| 6.1 | Click an agent card in the sidebar | Card highlights, floating detail panel appears over the map (left side) |
| 6.2 | Verify detail panel content | Shows type/class, domain, position, altitude, sensors section, stack section |
| 6.3 | Click the **X** close button on detail panel | Panel closes, agent deselected |
| 6.4 | Click a different agent in sidebar | Detail panel updates to new agent |
| 6.5 | Click an agent's map marker | Same agent selected, detail panel opens |
| 6.6 | Scroll the detail panel | Panel scrolls if content overflows (many sensors) |

---

## 7. Sensor Management — Add/Remove

| # | Action | Expected Result |
|---|--------|-----------------|
| 7.1 | Select an agent, click **+ Add Sensor** in detail panel | Buttons appear for available sensor types |
| 7.2 | Click **navsatfix** | Sensor card appears, toast shows success, sidebar sensor count updates |
| 7.3 | Add **imu**, **altimeter**, **twr_radio** | All 4 sensor cards visible in detail panel |
| 7.4 | Click **+ Add Sensor** again | Toast says "All sensor types already added" |
| 7.5 | Expand a sensor card, click **Remove Sensor** | Sensor removed, card disappears, count decreases |
| 7.6 | Verify via `ros2 topic list` | Removed sensor's topic is gone |
| 7.7 | Re-add the removed sensor | Sensor reappears, topic resumes publishing |

---

## 8. Sensor Details — Expandable Cards

| # | Action | Expected Result |
|---|--------|-----------------|
| 8.1 | Load scenario, select uav_01 | Detail panel shows 4 sensor cards (navsatfix, imu, altimeter, twr_radio) |
| 8.2 | Verify sensor card headers | Each shows sensor name (bold) and summary (msg type · rate Hz) |
| 8.3 | Click a sensor card header | Card expands showing topic path, all config params, noise params |
| 8.4 | Click the header again | Card collapses |
| 8.5 | Verify chevron animation | Arrow rotates 180° on expand, back on collapse |
| 8.6 | Verify topic path | Shows `/<agent_id>/<topic_suffix>` in accent color |
| 8.7 | Verify noise section | Noise params grouped under "noise" label with divider |

---

## 9. Sensor Config — Inline Editing

| # | Action | Expected Result |
|---|--------|-----------------|
| 9.1 | Expand a navsatfix sensor card | Values like `rate_hz`, `horizontal_std_m` are highlighted on hover |
| 9.2 | Click on the `rate_hz` value | Value turns into a text input, pre-filled with current value |
| 9.3 | Change the value to `10`, press Enter | Input closes, value updates to `10`, toast says "Updated navsatfix config" |
| 9.4 | Verify via `ros2 topic hz /<agent_id>/gps/fix` | Publishing rate changed to ~10 Hz |
| 9.5 | Click a noise value, change it, click away (blur) | Value saved on blur |
| 9.6 | Click a value, press Escape | Edit cancelled, original value restored |
| 9.7 | Verify `type` field is NOT editable | No hover highlight, clicking does nothing |
| 9.8 | Edit `max_range_m` on a twr_radio sensor | Value changes, backend applies new range limit |

---

## 10. Load Scenario

| # | Action | Expected Result |
|---|--------|-----------------|
| 10.1 | Click **Load** button in toolbar | Config modal opens with empty text area |
| 10.2 | Paste contents of `config/scenario.yaml`, click **Apply** | Modal closes, 5 agents appear (uav_01, uav_02, ugv_01, usv_01, uuv_01) |
| 10.3 | Verify agent markers | Correct shapes: 2 diamonds, 1 square, 1 pentagon, 1 inv triangle |
| 10.4 | Verify map auto-fits | Map zooms to show all 5 agents in the DC area |
| 10.5 | Select uav_01 | Detail panel: type UAV/quadrotor, domain 1, 4 sensor cards with full config |
| 10.6 | Select ugv_01 | Detail panel: type UGV/tracked, domain 3, 3 sensor cards |
| 10.7 | Select usv_01 | Detail panel: type USV/catamaran, domain 4, 3 sensor cards |
| 10.8 | Select uuv_01 | Detail panel: type UUV/torpedo, domain 5, 3 sensor cards |
| 10.9 | Check status badge | Should show **PAUSED** |
| 10.10 | Check sim time | Should show `T: 0.000s` |

---

## 11. Save Scenario

| # | Action | Expected Result |
|---|--------|-----------------|
| 11.1 | With agents loaded, click **Save** | Modal opens with YAML content |
| 11.2 | Verify YAML contains `sim_dt: 0.01` | Uses correct parameter name |
| 11.3 | Verify all 5 agents present with correct vehicle_type/class | Round-trip preserves all fields |
| 11.4 | Verify sensor configs are present | Each agent's sensors listed with noise params |
| 11.5 | Copy YAML, remove all agents, load the copied YAML | Agents restored identically |

---

## 12. Sim Controls — Play / Pause / Step

| # | Action | Expected Result |
|---|--------|-----------------|
| 12.1 | After loading scenario, click **Play** | Status badge changes to **RUNNING** (green), sim time starts advancing |
| 12.2 | Observe uav_02 on the map | Diamond marker moves along waypoint loop, shape rotates with heading |
| 12.3 | Observe ugv_01 | Square marker moves along triangular patrol route |
| 12.4 | Observe uuv_01 | Inverted triangle moves northeast along one-way path |
| 12.5 | Observe uav_01 and usv_01 | Markers stay put (static and commanded_velocity with no input) |
| 12.6 | Click **Pause** | Status badge changes to **PAUSED** (amber), all motion stops, sim time freezes |
| 12.7 | Note sim time, click **Step** | Time advances by exactly `sim_dt` (0.01s). Status stays **PAUSED** |
| 12.8 | Click **Step** 10 more times | Time advances by 0.01s each click |

---

## 13. Sim Controls — Speed

| # | Action | Expected Result |
|---|--------|-----------------|
| 13.1 | With sim paused, change speed to **2x**, click Play | Sim runs, time advances ~2x faster than wall-clock |
| 13.2 | Change to **0.5x** while running | Sim slows down, agents move at half speed |
| 13.3 | Change to **5x** | Agents move noticeably faster |
| 13.4 | Change to **10x** | Fast-forward. Waypoint agents complete loops quickly |
| 13.5 | Change to **Max** (0x) | Sim runs as fast as CPU allows |
| 13.6 | Change back to **1x** | Normal speed resumes |
| 13.7 | Verify via `ros2 topic hz /<agent_id>/gps/fix` | At 1x: ~5 Hz. At 2x: ~10 Hz. At 0.5x: ~2.5 Hz |

---

## 14. Sim Controls — Reset

| # | Action | Expected Result |
|---|--------|-----------------|
| 14.1 | Run sim at 1x for ~10 seconds, then Pause | Agents have moved from initial positions, sim time > 0 |
| 14.2 | Click **Reset** | Status badge shows **PAUSED**, sim time resets to `T: 0.000s` |
| 14.3 | Verify agent positions on map | All agents back at their initial positions |
| 14.4 | Click Play | Agents start motion patterns from the beginning |
| 14.5 | Run for 5s, reset, run again | Waypoint agents follow identical paths (deterministic seed) |

---

## 15. Map Dragging and Set Pose

| # | Action | Expected Result |
|---|--------|-----------------|
| 15.1 | With sim paused at time 0, drag an agent marker | Marker moves to new position (initial condition change) |
| 15.2 | Click Reset | Agent returns to the *dragged* position (updated initial pose) |
| 15.3 | Run sim for 5s, pause | Agents have moved |
| 15.4 | Drag a static agent (uav_01) to a new position | Marker moves (mid-run reposition) |
| 15.5 | Click Reset | Agent returns to its *original* initial pose (not the mid-run drag) |
| 15.6 | Click Play | Status goes to RUNNING |
| 15.7 | Try to drag an agent marker | Dragging should be disabled while running |
| 15.8 | Click Pause | Dragging re-enabled |

---

## 16. Scenario Events

| # | Action | Expected Result |
|---|--------|-----------------|
| 16.1 | Load scenario, set speed to 5x, click Play | Sim runs at 5x speed |
| 16.2 | Wait until T > 30s | `disable_sensor` on ugv_01/navsatfix — verify `/ugv_01/gps/fix` stops |
| 16.3 | Wait until T > 60s | `update_param` on uav_01/twr_radio `max_range_m` → 150 |
| 16.4 | Wait until T > 90s | `update_param` on uav_02/navsatfix `horizontal_std_m` → 10.0 (very noisy) |
| 16.5 | Wait until T > 120s | `set_pose` on usv_01 — marker should jump on the map |
| 16.6 | Wait until T > 180s | `update_param` on uuv_01/altimeter `std_m` → 2.0 |

---

## 17. TWR Radio Inter-Agent Ranging

| # | Action | Expected Result |
|---|--------|-----------------|
| 17.1 | Load scenario, run sim | Multiple agents have twr_radio sensors |
| 17.2 | `ros2 topic echo /uav_01/twr/ranges` | Ranges to nearby agents within 500m that also have twr_radio |
| 17.3 | As uav_02 flies, ranges from uav_01 change | Distance values increase/decrease |
| 17.4 | After T>60s (range reduced to 150m) | Some agents may drop out of uav_01's range list |

---

## 18. Ground Truth Publishing

| # | Action | Expected Result |
|---|--------|-----------------|
| 18.1 | Run sim, check `ros2 topic list` | Each agent has `/<agent_id>/sim/ground_truth` topic |
| 18.2 | `ros2 topic echo /uav_02/sim/ground_truth` | Shows lat/lon/alt changing with waypoint motion |
| 18.3 | `ros2 topic hz /uav_01/sim/ground_truth` | Should be ~50 Hz at 1x speed |
| 18.4 | Compare ground truth vs navsatfix for same agent | Ground truth exact; navsatfix has noise |

---

## 19. WebSocket Reconnect and State Sync

| # | Action | Expected Result |
|---|--------|-----------------|
| 19.1 | With agents added, kill and restart the WS bridge | WS indicator briefly Disconnected then Connected |
| 19.2 | Check agent list after reconnect | Agents still present (re-fetched via get_state) |
| 19.3 | Kill and restart the sim engine (backend restart) | Frontend detects stale backend, pushes UI state |
| 19.4 | Verify toast | "Backend restart detected — restoring state from UI" |

---

## 20. Agent Counter Recycling

| # | Action | Expected Result |
|---|--------|-----------------|
| 20.1 | Add uxv_01, uxv_02, uxv_03 via Place button | Three agents with sequential IDs |
| 20.2 | Remove uxv_01 | Agent removed |
| 20.3 | Place another UxV | Gets `uxv_01` (recycled lowest available) |
| 20.4 | Remove uxv_02 | Agent removed |
| 20.5 | Place another UxV | Gets `uxv_02` |
| 20.6 | Remove all agents, place one | Gets `uxv_01` |

---

## 21. Edge Cases

| # | Action | Expected Result |
|---|--------|-----------------|
| 21.1 | Load scenario twice in a row | Old agents cleared, new set loaded. No duplicates |
| 21.2 | Load scenario, add agent manually, load again | Manual agent removed, scenario agents restored |
| 21.3 | Load invalid YAML in config modal | Error toast, no crash |
| 21.4 | Remove a sensor that doesn't exist (race condition) | Graceful error, no crash |
| 21.5 | Rapidly click Play/Pause/Play/Pause | Status toggles cleanly, no stuck state |
| 21.6 | Change speed while paused, then play | New speed takes effect immediately |
| 21.7 | Reset with no agents loaded | No error, time resets to 0 |
| 21.8 | Step with no agents loaded | Time advances, no error |
| 21.9 | Place agent while sim is running | Agent added, appears on map (not draggable while running) |
| 21.10 | Remove agent while sim is running | Agent removed cleanly |
| 21.11 | Load scenario, save it, compare | Saved YAML should match loaded config |
| 21.12 | Disconnect network briefly, reconnect | WS auto-reconnects after 3s, state re-syncs |
| 21.13 | Edit sensor config while sim is running | Config applies immediately, sensor behavior changes |
| 21.14 | Hard-refresh browser with agents loaded | Agents re-sync from backend via get_state |
