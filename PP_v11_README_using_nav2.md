# ROSMASTER R2 (Yahboom) – ROS Melodic
## MBF + Regulated Pure Pursuit (ROS1 Port)
### Waypoints, Bag-to-Waypoints, and Side-by-Side with PP/FTG

This guide shows how to implement **Move Base Flex (MBF)** with the **Regulated Pure Pursuit (RPP)** controller (ROS1 back-port of the Nav2 controller) on a **Yahboom ROSMASTER R2** running **ROS Melodic**.

You will get:
- MBF + RPP navigation
- A Python waypoint follower (Melodic-safe)
- A rosbag → MBF waypoint converter
- A safe way to run MBF + RPP alongside your existing Pure Pursuit (PP) and FTG safety

---

## 0) What you are building (big picture)

Multiple navigation sources feed velocity commands:

- Teleop joystick → /cmd_vel_teleop
- Existing Pure Pursuit raceline → /cmd_vel_pp
- New MBF + RPP navigation → /cmd_vel_mbf

A **twist_mux** selects which one controls the robot and outputs **/cmd_vel**.

---

## 1) Assumptions

- ROS: Melodic  
- Workspace: ~/yahboomcar_ws (or ~/catkin_ws)  
- Topics: /scan, /odom, /cmd_vel  
- TF chain exists: map → odom → base_link  

---

## 2) Install prerequisites

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
  ros-melodic-rviz
```

---

## 3) Install Regulated Pure Pursuit (ROS1)

```bash
cd ~/yahboomcar_ws/src
git clone https://github.com/JohnTGZ/regulated_pure_pursuit_controller.git
cd ~/yahboomcar_ws
catkin_make
source devel/setup.bash
```

Verify plugin registration:

```bash
rospack plugins --attrib=plugin mbf_costmap_core::CostmapController
```

---

## 4) Create navigation package

```bash
cd ~/yahboomcar_ws/src
catkin_create_pkg r2_mbf_nav rospy roscpp std_msgs geometry_msgs nav_msgs tf tf2_ros actionlib mbf_msgs
cd ~/yahboomcar_ws
catkin_make
source devel/setup.bash
```

---

## 5) MBF configuration

Create `config/mbf_nav.yaml` and populate with controller, planner, and costmap settings.

---

## 6) Launch MBF safely

MBF publishes to `/cmd_vel_mbf`, not directly to motors.

---

## 7) Waypoint follower

A Python node sends sequential goals to MBF using an action client.

---

## 8) Bag to waypoint conversion

Convert recorded paths into YAML waypoint lists.

---

## 9) Side-by-side with PP/FTG

Use `twist_mux` to arbitrate between teleop, PP, and MBF.

---

## 10) Run order

1. Robot bringup
2. Localization
3. twist_mux
4. MBF
5. Waypoint follower

---

## 11) Debug checklist

- rostopic echo /cmd_vel_mbf
- rostopic echo /cmd_vel
- rosrun tf tf_echo map base_link

---

## 12) Folder layout

```
r2_mbf_nav/
 ├─ config/
 ├─ launch/
 ├─ scripts/
 └─ README.md
```

---

## Key takeaway

You now have **Nav2-style behavior on ROS Melodic**, without ROS2.
