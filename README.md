# hector_slam
Hector SLAM is an open-source ROS (Robot Operating System) package designed for 2D Simultaneous Localization and Mapping, specifically capable of building maps without requiring robot odometry
Perfect! Here's the complete README.md file ready to copy-paste directly into your GitHub repository:


### Step 1: Install ROS Noetic
`
- [Follow official ROS Noetic installation guide](https://wiki.ros.org/noetic/Installation/Ubuntu)
- [Hector SLAM Documentation](http://wiki.ros.org/hector_slam)



### Step 2: Install RPLIDAR Driver
```bash
cd ~/catkin_ws/src
git clone https://github.com/Slamtec/rplidar_ros.git
cd ~/catkin_ws
catkin_make
source devel/setup.bash
```

### Step 3: Install Hector SLAM
```bash
sudo apt-get install ros-noetic-hector-slam
```

### Step 4: Install Map Server (for saving maps)
```bash
sudo apt-get install ros-noetic-map-server
```

### Step 5: Verify Installation
```bash
rospack list | grep -E "rplidar_ros|hector"
```

---

## 🗺️ Understanding Frames & TF (Transform Library)

### What are Frames?
A **frame** is a coordinate system. Every sensor and robot part has its own frame.



  ![image_alt](https://github.com/hassanjizzine0-afk/hector_slam/blob/main/TF.png?raw=true) 


  
### Our Frame Setup
| Frame | Purpose |
|-------|---------|
| `map` | Fixed world reference frame |
| `laser` | LiDAR sensor frame |

### The Frame Chain
```
map ──────→ laser
  │            │
  │            └─ LiDAR sensor position
  └─ Global world reference (fixed)
```

### Why Static Transform?
```bash
rosrun tf static_transform_publisher 0 0 0 0 0 0 map laser 100
```

**This tells ROS:** The `laser` frame is located at the exact same position and orientation as the `map` frame.

**The 6 numbers mean:**
```
Position: 0 0 0    → X, Y, Z offset (meters)
Rotation: 0 0 0    → Yaw, Pitch, Roll (radians)
```

---

## ⚙️ Parameters Explained

### Frame Parameters
| Parameter | Default | Our Value | Why |
|-----------|---------|-----------|-----|
| `_map_frame` | `map_link` | `map` | World reference frame |
| `_base_frame` | `base_link` | `laser` | LiDAR is our robot center |
| `_odom_frame` | `odom` | `laser` | No wheel odometry available |

### Map Building Parameters
| Parameter | Default | Our Value | Why |
|-----------|---------|-----------|-----|
| `_map_resolution` | `0.025` | `0.05` | 5cm per pixel - good balance |
| `_map_size` | `1024` | `2048` | 102 meter square map |

### Update Thresholds (CRITICAL!)
| Parameter | Default | Our Value | Why Change? |
|-----------|---------|-----------|-------------|
| `_map_update_distance_thresh` | `0.4 m` | `0.05 m` | Update after only 5cm movement |
| `_map_update_angle_thresh` | `0.9 rad` (51°) | `0.05 rad` (3°) | Update after small rotations |

> **⚠️ This is why your map wasn't updating!** Default thresholds are too high for small movements.

### Laser Filter Parameters
| Parameter | Default | Our Value | Why |
|-----------|---------|-----------|-----|
| `_laser_min_dist` | `0.4 m` | `0.4 m` | Ignore objects closer than 40cm |
| `_laser_max_dist` | `30.0 m` | `5.0 m` | RPLIDAR A1 max range is 6m |
| `_scan_topic` | `/scan` | `/scan` | Default laser topic name |

---

## 🚀 Running the System (5 Terminals)

### Prerequisite (Once per boot)
```bash
sudo chmod 666 /dev/ttyUSB0
```

---

### Terminal 1: ROS Core
```bash
roscore
```
> **Keep this running!** This is the ROS master.

---

### Terminal 2: RPLIDAR Driver
```bash

rosrun rplidar_ros rplidarNode \
    _serial_port:=/dev/ttyUSB0 \
    _serial_baudrate:=115200 \
    _frame_id:=laser \
    _inverted:=false \
    _angle_compensate:=true
```
> **Keep this running!** Publishes `/scan` topic with LiDAR data.

---

### Terminal 3: Static Transform (Connects frames)
```bash

rosrun tf static_transform_publisher 0 0 0 0 0 0 map laser 100
```
> **Keep this running!** No output is normal.

---

### Terminal 4: Hector SLAM Node
```bash

rosrun hector_mapping hector_mapping \
    _map_frame:=map \
    _base_frame:=laser \
    _odom_frame:=laser \
    _scan_topic:=/scan \
    _map_resolution:=0.05 \
    _map_size:=2048 \
    _map_update_distance_thresh:=0.05 \
    _map_update_angle_thresh:=0.05 \
    _laser_max_dist:=5.0
```
> **Keep this running!** Builds the map from LiDAR data.

---

### Terminal 5: RViz Visualization
```bash
rviz
```

---

## 🎨 RViz Configuration (CRITICAL!)

Once RViz opens, do these steps **IN ORDER**:

### Step 1: Set Fixed Frame
```
Left panel → Global Options → Fixed Frame
Type: map (press Enter)
```

### Step 2: Add Laser Scan
```
Click "Add" button (bottom left)
→ By topic tab
→ Find "/scan"
→ Select "LaserScan"
→ Click "OK"
```

### Step 3: Add Map
```
Click "Add" button
→ By topic tab
→ Find "/map"
→ Select "Map"
→ Click "OK"
```

### Step 4: Adjust View
- Zoom in/out with mouse scroll
- Pan with right-click drag
- Rotate with left-click drag

---

## 💾 Saving the Map

When you have built a good map:

```bash
# In a new terminal
source /opt/ros/noetic/setup.bash
rosrun map_server map_saver -f ~/my_map
```

**Files created:**
- `my_map.pgm` - Image file (viewable in any image viewer)
- `my_map.yaml` - Configuration file

---

## 🔄 Quick Summary - All Terminals

| Terminal | Command | Purpose |
|----------|---------|---------|
| 1 | `roscore` | ROS master |
| 2 | `rplidarNode` | Read LiDAR hardware |
| 3 | `static_transform_publisher` | Connect map→laser frames |
| 4 | `hector_mapping` | Build map from laser data |
| 5 | `rviz` | Visualize the map |

---

## ❓ Troubleshooting

| Problem | Solution |
|---------|----------|
| Permission denied on `/dev/ttyUSB0` | `sudo chmod 666 /dev/ttyUSB0` |
| Map not updating | Lower `map_update_distance_thresh` and `map_update_angle_thresh` |
| No laser scan in RViz | Check Fixed Frame = `map` |
| Hector SLAM not found | Run `source /opt/ros/noetic/setup.bash` |
| Map is empty (all gray) | Move the LiDAR around! |
| "Failed to contact master" | roscore not running (Terminal 1) |

---

## 📊 How to Know It's Working

### Check Commands
```bash
# Check LiDAR is publishing
rostopic hz /scan            # Should show 5-10 Hz

# Check map is publishing  
rostopic hz /map              # Should show 0.5-1 Hz

# Check nodes are running
rosnode list                  # Should show: /hector_mapping, /rplidar_node

# Check frame connection
rosrun tf view_frames
evince frames.pdf             # Should show map→laser connection
```

### In RViz You Should See:
- ✅ Gray grid (the map frame)
- ✅ Red dots (laser scan data)
- ✅ Black/white squares (map being built)

---

## 🎯 Key Insights

1. **Map only updates when LiDAR MOVES** - You must walk around!
2. **Lower thresholds = More sensitive updates** - Default 0.4m is too high
3. **Frame names MUST match** - `_frame_id:=laser` in LiDAR matches `map laser` in TF
4. **Static transform tells ROS where frames are** - Without it, nothing works!

---

## 📚 References
- [Hector SLAM Documentation](http://wiki.ros.org/hector_slam)
- [RPLIDAR ROS Package](http://wiki.ros.org/rplidar)
- [TF Documentation](http://wiki.ros.org/tf)

---

