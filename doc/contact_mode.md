# Contact Mode Implementation Details

This document explains the technical implementation of contact mode in the Zordi fork of xarm_ros2.

## Overview

Contact mode disables xArm safety features to allow contact-rich manipulation (insertion, polishing, assembly). Settings are applied at driver startup and stored in RAM only (not persisted to controller).

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Controller Persistent Storage (survives power cycles)      │
│  ← Written by: Web UI, save_conf(), or clean_conf()         │
└─────────────────────────────────────────────────────────────┘
                            ↓
                    Robot powers on
                            ↓
              Uses saved settings (from Web UI)
                            ↓
                   ROS driver starts
                            ↓
           ┌─────────────────────────────────────┐
           │  override_webui = false (default)   │
           │  → Keep Web UI settings             │
           ├─────────────────────────────────────┤
           │  override_webui = true              │
           │  → Apply contact mode (RAM only)    │
           └─────────────────────────────────────┘
                            ↓
                    Driver stops
                            ↓
              Reverts to saved settings (Web UI)
```

## Parameters

### Master Switch

| Parameter | Default | Description |
|-----------|---------|-------------|
| `override_webui` | false | If true, apply contact mode settings; if false, use Web UI |

### Contact Mode Settings (applied when override_webui=true)

| Parameter | Factory | Contact Mode | Description |
|-----------|---------|--------------|-------------|
| `check_tcp_limit` | true | **false** | SDK rejects commands outside TCP limits |
| `check_joint_limit` | true | **false** | SDK rejects commands outside joint limits |
| `collision_sensitivity` | 3 | **0** | External collision detection (0=off, 1-5) |
| `collision_rebound` | true | **false** | Retract on collision |
| `self_collision_detection` | true | **false** | Prevent arm hitting itself |
| `reduced_mode` | true | **false** | Limit speed/workspace |
| `ft_collision_detection` | false | **false** | F/T sensor triggers collision stop |
| `ft_collision_rebound` | false | **false** | F/T sensor triggers retract |

### Not Exposed as Launch Parameters

| Setting | Factory | Description |
|---------|---------|-------------|
| `fence_mode` | true | Workspace boundary (±200mm cube) |

## Implementation

### Files Modified

| File | Changes |
|------|---------|
| `xarm_api/src/xarm_driver.cpp` | Read saved settings, apply overrides, verification logging |
| `xarm_api/launch/_robot_driver.launch.py` | Expose all contact mode launch arguments |
| `xarm_api/config/xarm_params.yaml` | Enable collision services for runtime changes |

### Driver Startup Flow

1. Connect to robot
2. Read and log saved settings from controller (Web UI)
3. If `override_webui=true`:
   - Apply all contact mode settings
   - Log what was applied
4. Verify settings were applied correctly
5. Log verification summary

### Startup Logs

With `override_webui=true`:
```
[INFO] === SAVED SETTINGS (from Web UI) ===
[INFO]   collision_sensitivity=3, reduced_mode=ON, fence_mode=ON, collision_rebound=ON
[INFO]   ft_collision_detection=OFF, ft_collision_rebound=OFF
[INFO] =====================================
[INFO] override_webui=true: Applying contact mode overrides...
[INFO] Collision settings applied: sensitivity=0, rebound=0, self_collision=0
[INFO] F/T collision settings applied: detection=0, rebound=0
[INFO] Reduced mode: OFF (speed limits disabled)
[INFO] === CONTACT MODE VERIFICATION ===
[INFO]   SDK pre-checks: check_tcp_limit=0, check_joint_limit=0
[INFO]   Collision (robot reports): sensitivity=0 (want 0)
[INFO]   ✓ CONTACT MODE ACTIVE: All safety limits disabled
[INFO] =================================
```

With `override_webui=false` (default):
```
[INFO] === SAVED SETTINGS (from Web UI) ===
[INFO]   collision_sensitivity=3, reduced_mode=ON, fence_mode=ON, collision_rebound=ON
[INFO] =====================================
[INFO] override_webui=false: Using Web UI settings (no overrides applied)
```

## Persistence

Settings changed via the driver are RAM-only and revert when:
- Driver stops
- Robot power cycles

To persist settings:
```bash
# Persist current settings to controller (disabled by default)
ros2 service call /xarm/save_conf xarm_msgs/srv/Call

# Restore factory defaults (disabled by default)
ros2 service call /xarm/clean_conf xarm_msgs/srv/Call
```

Enable these services in `xarm_params.yaml`:
```yaml
services:
  save_conf: true
  clean_conf: true
```

## Runtime Services

These services are enabled by default for runtime adjustments:

```bash
# Collision detection
ros2 service call /xarm/set_collision_sensitivity xarm_msgs/srv/SetInt16 "{data: 0}"
ros2 service call /xarm/set_collision_rebound xarm_msgs/srv/SetInt16 "{data: 0}"
ros2 service call /xarm/set_self_collision_detection xarm_msgs/srv/SetInt16 "{data: 0}"

# Reduced mode
ros2 service call /xarm/set_reduced_mode xarm_msgs/srv/SetInt16 "{data: 0}"
```

## Web UI Limitation

The xArm Web UI may only allow `collision_sensitivity` values 1-5. The SDK can set 0 (completely off), which is why `override_webui=true` is required for full contact mode.

## Safety Considerations

| Disabled Feature | Risk | Mitigation |
|------------------|------|------------|
| Collision sensitivity | Arm won't stop on impacts | Conservative speeds, e-stop |
| Collision rebound | No automatic retraction | Software limits |
| Self-collision | Arm can hit itself | Collision-free paths |
| Reduced mode | Full speed available | Conservative velocities |
| F/T collision | F/T sensor won't trigger stops | External e-stop |

## Related

- [xarm_ft.md](../../xarm_ft.md) — F/T sensor modes and Error 52 handling

