# ROSMASTER R2 (Yahboom) – ROS Melodic
## Full Manual: MBF + Regulated Pure Pursuit (ROS1 Port) + Waypoints + Bag→Waypoints + Side‑by‑Side with PP/FTG

This is a **complete, drop‑in manual** for adding **Move Base Flex (MBF)** + a **Regulated Pure Pursuit (RPP) controller plugin (ROS1)** to a **Yahboom ROSMASTER R2** running **ROS1 Melodic**, including:

✅ A **simple waypoint follower node** (Python, Melodic‑safe)  
✅ A **rosbag → MBF waypoints YAML** converter  
✅ A safe way to run **MBF+RPP side‑by‑side** with your existing **Pure Pursuit (PP) + FTG Safety** using `twist_mux`  
✅ Full folder layout + full file contents + commands

> Important: “Back‑port / ROS1 port” means the controller’s **logic** is inspired by Nav2, but it runs **100% on ROS1**. No ROS2/Nav2 runtime is used.

---

# Table of Contents

1. Safety First  
2. What You Are Building (Big Picture)  
3. Prereqs and Install (Melodic)  
4. Add the RPP Controller Plugin (ROS1)  
5. Create a Small “Glue” Package (`r2_mbf_nav`)  
6. MBF Config (`mbf_nav.yaml`)  
7. MBF Launch (`mbf_nav.launch`)  
8. Waypoint Follower (Python Node)  
9. Raceline Bag → MBF Waypoints (Python Tool)  
10. Side‑by‑Side with PP/FTG using `twist_mux`  
11. Recommended Run Order  
12. RViz Setup (Goal Sending + Path Display)  
13. Troubleshooting Checklist (Most Common Failures)  
14. Tuning Starter Notes (R2‑scale values)  
15. Folder Layout (Final)  

---

# 1) Safety First

- First tests: **robot on a stand** (wheels off the ground).
- Always have a **kill method**: joystick deadman, E‑stop node, RC switch, or power switch.
- Start with **low speeds**.
- Test in an **open area** away from people/pets/stairs/traffic.

---

# 2) What You Are Building (Big Picture)

You will have **multiple “drivers”** that can output velocity commands:

- Teleop joystick → `/cmd_vel_teleop`
- Your existing Pure Pursuit raceline → `/cmd_vel_pp`
- New MBF navigation (RViz goals → global plan → RPP controller) → `/cmd_vel_mbf`

Then `twist_mux` chooses which one wins and outputs **only one** final command:

- `twist_mux` output → `/cmd_vel` (the motor driver listens here)

```
/cmd_vel_teleop  ----\
/cmd_vel_pp      ----- >  twist_mux  ---->  /cmd_vel  ---->  R2 base driver
/cmd_vel_mbf     ----/
```

If you already use an FTG safety node that outputs a “safe cmd” topic, we’ll show a safe pattern for that too.

---

# 3) Prereqs and Install (ROS Melodic)

## 3.1 Confirm you are on Melodic

```bash
echo $ROS_DISTRO
# should print: melodic
```

## 3.2 Install required packages

```bash
sudo apt update

sudo apt install -y \
  ros-melodic-move-base-flex \
  ros-melodic-navigation \
  ros-melodic-costmap-2d \
  ros-melodic-tf2-ros \
  ros-melodic-tf2-geometry-msgs \
  ros-melodic-ddynamic-reconfigure \
  ros-melodic-joy \
  ros-melodic-twist-mux \
  ros-melodic-rviz \
  python-yaml
```

> If you already have `navigation` installed, that’s fine.

---

# 4) Add the Regulated Pure Pursuit Controller (ROS1)

This is the ROS1 “port/back‑port” controller plugin.

## 4.1 Clone into your workspace

Choose the workspace you actually use on the R2. Example uses `~/yahboomcar_ws`.

```bash
cd ~/yahboomcar_ws/src
git clone https://github.com/JohnTGZ/regulated_pure_pursuit_controller.git
```

## 4.2 Build

```bash
cd ~/yahboomcar_ws
catkin_make
source devel/setup.bash
```

## 4.3 Verify plugin registration (IMPORTANT)

This prints the plugin “type string” MBF will load.

```bash
rospack plugins --attrib=plugin mbf_costmap_core::CostmapController
```

You should see something mentioning regulated pure pursuit.
**Copy the exact `type` string** shown.  
In many setups it is:

```
regulated_pure_pursuit_controller/RegulatedPurePursuitController
```

…but **use exactly what your robot prints**.

---

# 5) Create a Small Glue Package: `r2_mbf_nav`

This package holds:
- MBF configs
- Launch files
- Waypoint scripts
- Bag conversion scripts

## 5.1 Create the package

```bash
cd ~/yahboomcar_ws/src
catkin_create_pkg r2_mbf_nav rospy roscpp std_msgs geometry_msgs nav_msgs tf tf2_ros actionlib mbf_msgs
```

## 5.2 Create folders

```bash
mkdir -p ~/yahboomcar_ws/src/r2_mbf_nav/{config,launch,scripts}
```

## 5.3 Build again

```bash
cd ~/yahboomcar_ws
catkin_make
source devel/setup.bash
```

---

# 6) MBF Configuration File (Complete)

Create:

`~/yahboomcar_ws/src/r2_mbf_nav/config/mbf_nav.yaml`

> **Note:** Replace controller `type:` with your `rospack plugins ...` output if different.

```yaml
# =========================
# MBF basic loop rates
# =========================
controller_frequency: 10.0
planner_frequency: 1.0

# =========================
# Plugins (MBF)
# =========================
controllers:
  - name: RPP
    type: regulated_pure_pursuit_controller/RegulatedPurePursuitController  # <-- replace if your plugin string differs

planners:
  - name: NavFn
    type: navfn/NavfnROS

# =========================
# Controller params (starter values for small Ackermann-ish robots)
# Namespace must match controller name: "RPP:"
# =========================
RPP:
  desired_linear_vel: 0.25
  lookahead_dist: 0.60
  min_lookahead_dist: 0.30
  max_lookahead_dist: 1.20

  use_velocity_scaled_lookahead_dist: true
  approach_velocity_scaling_dist: 1.0
  use_approach_linear_velocity_scaling: true

  # Limit turning aggression
  max_angular_vel: 1.2
  rotate_to_heading_angular_vel: 1.0
  rotate_to_heading_min_angle: 0.5

  transform_tolerance: 0.2

# =========================
# Costmaps (simple starter)
# You can later reuse your existing move_base costmap YAML
# =========================

global_costmap:
  global_frame: map
  robot_base_frame: base_link
  update_frequency: 5.0
  publish_frequency: 2.0
  static_map: true
  rolling_window: false
  transform_tolerance: 0.3

local_costmap:
  global_frame: odom
  robot_base_frame: base_link
  update_frequency: 10.0
  publish_frequency: 5.0
  static_map: false
  rolling_window: true
  width: 6.0
  height: 6.0
  resolution: 0.05
  transform_tolerance: 0.3
```

---

# 7) MBF Launch File (Complete)

Create:

`~/yahboomcar_ws/src/r2_mbf_nav/launch/mbf_nav.launch`

This runs MBF but publishes to **`/cmd_vel_mbf`** (NOT `/cmd_vel`).

```xml
<launch>
  <!-- Load MBF + costmap + controller params -->
  <rosparam command="load" file="$(find r2_mbf_nav)/config/mbf_nav.yaml" />

  <!-- MBF Costmap Navigation -->
  <node pkg="mbf_costmap_nav" type="mbf_costmap_nav" name="move_base_flex" output="screen">
    <!-- Adjust these if your topics differ -->
    <remap from="scan" to="/scan"/>
    <remap from="odom" to="/odom"/>

    <!-- IMPORTANT: publish to a dedicated topic -->
    <remap from="cmd_vel" to="/cmd_vel_mbf"/>
  </node>
</launch>
```

---

# 8) Simple Waypoint Follower Node (Python, Melodic‑safe)

This node sends **multiple goals in order** to MBF.
It reads a YAML file like:

```yaml
waypoints:
  - {x: 0.5, y: 0.0, yaw: 0.0}
  - {x: 1.2, y: 0.2, yaw: 0.0}
```

## 8.1 Waypoint YAML (example)

Create:

`~/yahboomcar_ws/src/r2_mbf_nav/config/waypoints_demo.yaml`

```yaml
waypoints:
  - {x: 0.5, y: 0.0, yaw: 0.0}
  - {x: 1.2, y: 0.2, yaw: 0.0}
  - {x: 2.0, y: 0.6, yaw: 1.57}
```

## 8.2 The follower script (complete)

Create:

`~/yahboomcar_ws/src/r2_mbf_nav/scripts/waypoint_follower_mbf.py`

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import yaml
import rospy
import actionlib
import tf

from geometry_msgs.msg import PoseStamped, Quaternion
from mbf_msgs.msg import MoveBaseAction, MoveBaseGoal

def yaw_to_quat(yaw):
    q = tf.transformations.quaternion_from_euler(0.0, 0.0, yaw)
    return Quaternion(x=q[0], y=q[1], z=q[2], w=q[3])

def load_waypoints(yaml_path):
    with open(yaml_path, 'r') as f:
        data = yaml.safe_load(f)
    wps = data.get('waypoints', [])
    if not wps:
        raise RuntimeError("No waypoints found in YAML: %s" % yaml_path)
    return wps

def main():
    rospy.init_node("waypoint_follower_mbf")

    yaml_path = rospy.get_param("~waypoints_yaml", "")
    frame_id  = rospy.get_param("~frame_id", "map")
    planner   = rospy.get_param("~planner", "NavFn")
    controller= rospy.get_param("~controller", "RPP")
    wait_s    = float(rospy.get_param("~wait_for_server_s", 10.0))
    pause_s   = float(rospy.get_param("~goal_pause_s", 1.0))

    if not yaml_path:
        rospy.logerr("Missing ~waypoints_yaml. Example:\n"
                     "rosrun r2_mbf_nav waypoint_follower_mbf.py _waypoints_yaml:=/path/to/waypoints.yaml")
        return

    waypoints = load_waypoints(yaml_path)

    client = actionlib.SimpleActionClient("move_base_flex/move_base", MoveBaseAction)
    rospy.loginfo("Waiting for MBF action server: move_base_flex/move_base ...")
    if not client.wait_for_server(rospy.Duration(wait_s)):
        rospy.logerr("MBF action server not available. Is mbf_nav.launch running?")
        return

    rospy.loginfo("Loaded %d waypoints from %s", len(waypoints), yaml_path)

    for i, wp in enumerate(waypoints):
        x = float(wp.get('x', 0.0))
        y = float(wp.get('y', 0.0))
        yaw = float(wp.get('yaw', 0.0))

        goal = MoveBaseGoal()
        goal.target_pose = PoseStamped()
        goal.target_pose.header.stamp = rospy.Time.now()
        goal.target_pose.header.frame_id = frame_id
        goal.target_pose.pose.position.x = x
        goal.target_pose.pose.position.y = y
        goal.target_pose.pose.orientation = yaw_to_quat(yaw)

        goal.planner = planner
        goal.controller = controller

        rospy.loginfo("Sending waypoint %d/%d: x=%.3f y=%.3f yaw=%.3f",
                      i+1, len(waypoints), x, y, yaw)

        client.send_goal(goal)
        client.wait_for_result()

        state = client.get_state()
        # actionlib SUCCEEDED == 3
        if state != 3:
            rospy.logwarn("Waypoint %d FAILED (state=%d). Stopping.", i+1, state)
            break

        rospy.loginfo("Waypoint %d reached. Pause %.1fs", i+1, pause_s)
        rospy.sleep(pause_s)

    rospy.loginfo("Waypoint follower complete.")

if __name__ == "__main__":
    main()
```

Make it executable:

```bash
chmod +x ~/yahboomcar_ws/src/r2_mbf_nav/scripts/waypoint_follower_mbf.py
```

## 8.3 Run the follower

Terminal A (MBF):
```bash
source ~/yahboomcar_ws/devel/setup.bash
roslaunch r2_mbf_nav mbf_nav.launch
```

Terminal B (waypoints):
```bash
source ~/yahboomcar_ws/devel/setup.bash
rosrun r2_mbf_nav waypoint_follower_mbf.py \
  _waypoints_yaml:=`rospack find r2_mbf_nav`/config/waypoints_demo.yaml
```

---

# 9) Convert Bag‑Recorded Raceline → MBF Waypoints YAML (Complete Tool)

This tool reads a bag and tries:
1) First `nav_msgs/Path` topic, else
2) First `geometry_msgs/PoseStamped` topic

Then downsamples points by distance and writes a YAML list of `{x,y,yaw}`.

## 9.1 Create converter script

Create:

`~/yahboomcar_ws/src/r2_mbf_nav/scripts/bag_to_mbf_waypoints.py`

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import math
import yaml
import argparse
import rosbag

def quat_to_yaw(q):
    # yaw from quaternion
    siny_cosp = 2.0 * (q.w*q.z + q.x*q.y)
    cosy_cosp = 1.0 - 2.0 * (q.y*q.y + q.z*q.z)
    return math.atan2(siny_cosp, cosy_cosp)

def dist(a, b):
    return math.hypot(a[0]-b[0], a[1]-b[1])

def pick_topics(bag):
    info = bag.get_type_and_topic_info()[1]
    path_topics = []
    pose_topics = []
    for t, meta in info.items():
        if meta.msg_type == 'nav_msgs/Path':
            path_topics.append(t)
        if meta.msg_type == 'geometry_msgs/PoseStamped':
            pose_topics.append(t)
    return path_topics, pose_topics

def extract_points_from_path(msg):
    pts = []
    for ps in msg.poses:
        x = ps.pose.position.x
        y = ps.pose.position.y
        yaw = quat_to_yaw(ps.pose.orientation)
        pts.append((x, y, yaw))
    return pts

def extract_points_from_pose_stream(bag, topic, max_msgs=None):
    pts = []
    count = 0
    for _, msg, _ in bag.read_messages(topics=[topic]):
        x = msg.pose.position.x
        y = msg.pose.position.y
        yaw = quat_to_yaw(msg.pose.orientation)
        pts.append((x, y, yaw))
        count += 1
        if max_msgs and count >= max_msgs:
            break
    return pts

def downsample_by_distance(pts, min_sep):
    if not pts:
        return pts
    out = [pts[0]]
    last = (pts[0][0], pts[0][1])
    for x, y, yaw in pts[1:]:
        if dist((x, y), last) >= min_sep:
            out.append((x, y, yaw))
            last = (x, y)
    return out

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("bag", help="input .bag file")
    ap.add_argument("--out", default="mbf_waypoints.yaml", help="output yaml")
    ap.add_argument("--topic", default="", help="force a topic")
    ap.add_argument("--min_sep", type=float, default=0.50, help="min distance between waypoints (m)")
    ap.add_argument("--max_msgs", type=int, default=0, help="limit pose stream messages (0 = no limit)")
    args = ap.parse_args()

    bag = rosbag.Bag(args.bag, 'r')
    path_topics, pose_topics = pick_topics(bag)

    chosen = args.topic
    pts = []

    if chosen:
        msg_type = bag.get_type_and_topic_info()[1][chosen].msg_type
        if msg_type == 'nav_msgs/Path':
            for _, msg, _ in bag.read_messages(topics=[chosen]):
                pts = extract_points_from_path(msg)
                break
        elif msg_type == 'geometry_msgs/PoseStamped':
            pts = extract_points_from_pose_stream(bag, chosen, args.max_msgs if args.max_msgs > 0 else None)
        else:
            raise RuntimeError("Unsupported msg type on %s: %s" % (chosen, msg_type))
    else:
        if path_topics:
            chosen = path_topics[0]
            for _, msg, _ in bag.read_messages(topics=[chosen]):
                pts = extract_points_from_path(msg)
                break
        elif pose_topics:
            chosen = pose_topics[0]
            pts = extract_points_from_pose_stream(bag, chosen, args.max_msgs if args.max_msgs > 0 else None)
        else:
            raise RuntimeError("No nav_msgs/Path or geometry_msgs/PoseStamped topics found in bag.")

    bag.close()

    pts_ds = downsample_by_distance(pts, args.min_sep)

    out = {"waypoints": []}
    for x, y, yaw in pts_ds:
        out["waypoints"].append({"x": float(x), "y": float(y), "yaw": float(yaw)})

    with open(args.out, 'w') as f:
        yaml.safe_dump(out, f, default_flow_style=False)

    print("Chosen topic:", chosen)
    print("Raw points:", len(pts))
    print("Downsampled waypoints:", len(pts_ds))
    print("Wrote:", args.out)

if __name__ == "__main__":
    main()
```

Make it executable:

```bash
chmod +x ~/yahboomcar_ws/src/r2_mbf_nav/scripts/bag_to_mbf_waypoints.py
```

## 9.2 Use the converter

First inspect your bag:

```bash
rosbag info ~/bags/my_raceline.bag
```

Convert automatically:

```bash
source ~/yahboomcar_ws/devel/setup.bash

rosrun r2_mbf_nav bag_to_mbf_waypoints.py \
  ~/bags/my_raceline.bag \
  --out `rospack find r2_mbf_nav`/config/waypoints_from_bag.yaml \
  --min_sep 0.50
```

If you want to force the topic:

```bash
rosrun r2_mbf_nav bag_to_mbf_waypoints.py \
  ~/bags/my_raceline.bag \
  --topic /your_topic_here \
  --out `rospack find r2_mbf_nav`/config/waypoints_from_bag.yaml \
  --min_sep 0.50
```

---

# 10) Run MBF + RPP Side‑by‑Side with PP/FTG (Safe `twist_mux` Pattern)

## 10.1 Rule
**Never** let multiple nodes publish directly to `/cmd_vel`.

Instead:
- MBF publishes `/cmd_vel_mbf`
- PP publishes `/cmd_vel_pp`
- Teleop publishes `/cmd_vel_teleop`
- `twist_mux` outputs `/cmd_vel` (only one publisher)

## 10.2 `twist_mux` config (complete)

Create:

`~/yahboomcar_ws/src/r2_mbf_nav/config/twist_mux.yaml`

```yaml
topics:
  - name: teleop
    topic: /cmd_vel_teleop
    timeout: 0.5
    priority: 100

  - name: pure_pursuit
    topic: /cmd_vel_pp
    timeout: 0.5
    priority: 80

  - name: mbf_nav
    topic: /cmd_vel_mbf
    timeout: 0.5
    priority: 70
```

## 10.3 `twist_mux` launch (complete)

Create:

`~/yahboomcar_ws/src/r2_mbf_nav/launch/twist_mux.launch`

```xml
<launch>
  <node pkg="twist_mux" type="twist_mux" name="twist_mux" output="screen">
    <rosparam command="load" file="$(find r2_mbf_nav)/config/twist_mux.yaml"/>
    <remap from="cmd_vel_out" to="/cmd_vel"/>
  </node>
</launch>
```

## 10.4 Remap your PP output (example)
If your PP node currently publishes `/cmd_vel`, remap it to `/cmd_vel_pp` in its launch:

```xml
<remap from="/cmd_vel" to="/cmd_vel_pp"/>
```

(Exact remap depends on your PP node.)

## 10.5 If FTG Safety should be final authority (recommended)
If your FTG safety node takes an input cmd and outputs a safe cmd, do this:

- `twist_mux` output → `/cmd_vel_mux`
- FTG safety input → `/cmd_vel_mux`
- FTG safety output → `/cmd_vel` (motor driver listens here)

That makes **FTG safety always the last gate**.

---

# 11) Recommended Run Order (What to launch, in order)

## A) Bring up robot drivers + sensors first
You need:
- Motor driver running
- `/scan` publishing
- `/odom` publishing
- TF for `odom -> base_link`

## B) Localization / Map
For navigation on a saved map:
- `map_server` + `amcl`
(or Cartographer localization mode)

You must have:
- `map -> odom` transform
- so final chain is `map -> odom -> base_link`

## C) Start twist_mux (so teleop always can take over)

```bash
source ~/yahboomcar_ws/devel/setup.bash
roslaunch r2_mbf_nav twist_mux.launch
```

## D) Start MBF

```bash
source ~/yahboomcar_ws/devel/setup.bash
roslaunch r2_mbf_nav mbf_nav.launch
```

## E) Test with RViz “2D Nav Goal”
Or run waypoint follower (Section 8).

---

# 12) RViz Setup (Goals + Show Path)

## 12.1 Fixed frame
- Set **Fixed Frame** = `map`

## 12.2 Send a goal
- Click **2D Nav Goal**, then click on the map and drag to set orientation.

## 12.3 Show global path
- Add display: **Path**
- Choose the planner’s path topic.
  Common possibilities include:
  - `/move_base_flex/NavFn/plan`
  - `/move_base_flex/NavFnROS/plan`
  - `/move_base_flex/plan`

If you aren’t sure, list candidates:

```bash
rostopic list | grep -i plan
```

---

# 13) Troubleshooting Checklist (Most Common Failures)

## 13.1 “MBF runs but robot won’t move”
Check if MBF is publishing:

```bash
rostopic echo /cmd_vel_mbf
```

Check if twist_mux is outputting:

```bash
rostopic echo /cmd_vel
```

Check who publishes `/cmd_vel`:

```bash
rostopic info /cmd_vel
```

You want:
- Publisher = `twist_mux` (or FTG final node)
- NOT multiple publishers fighting

## 13.2 “Goal accepted but planner fails”
Usually map/localization problem.
Verify TF chain:

```bash
rosrun tf tf_echo map base_link
```

If this doesn’t print smoothly, fix localization / TF.

## 13.3 “Costmap does not see obstacles”
Verify scan:

```bash
rostopic echo -n 1 /scan
```

Verify the scan frame exists in TF:

```bash
rosrun tf tf_echo base_link <scan_frame_here>
```

## 13.4 “Robot rotates wildly / oversteers”
Reduce:
- `desired_linear_vel`
- `max_angular_vel`
Increase:
- `lookahead_dist` slightly

---

# 14) Tuning Starter Notes (R2‑scale)

Start conservative:

- `desired_linear_vel`: 0.20–0.30
- `lookahead_dist`: 0.5–0.8
- `max_angular_vel`: 0.8–1.2

If robot “cuts corners” too much:
- increase `lookahead_dist`
If robot “wiggles”:
- reduce `max_angular_vel`
- reduce `desired_linear_vel`

---

# 15) Final Folder Layout (What you should have)

```text
~/yahboomcar_ws/src/
  regulated_pure_pursuit_controller/
  r2_mbf_nav/
    config/
      mbf_nav.yaml
      twist_mux.yaml
      waypoints_demo.yaml
      waypoints_from_bag.yaml        (generated)
    launch/
      mbf_nav.launch
      twist_mux.launch
    scripts/
      waypoint_follower_mbf.py
      bag_to_mbf_waypoints.py
    README.md
```

---

# Quick Start (Copy/Paste)

```bash
# Build everything
cd ~/yahboomcar_ws
catkin_make
source devel/setup.bash

# Start twist_mux
roslaunch r2_mbf_nav twist_mux.launch

# Start MBF
roslaunch r2_mbf_nav mbf_nav.launch

# Run waypoint follower
rosrun r2_mbf_nav waypoint_follower_mbf.py \
  _waypoints_yaml:=`rospack find r2_mbf_nav`/config/waypoints_demo.yaml
```

---

## If you want this to be 100% drop‑in on your R2
Run these and paste outputs:

```bash
rostopic list | egrep "cmd_vel|scan|odom"
rostopic info /cmd_vel
rostopic info /scan
rostopic info /odom
rosrun tf tf_echo map base_link
```

Then the only “customization” left is correct remaps for Yahboom topic names.
