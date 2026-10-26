## Update Ubuntu and install prerequisites
```
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl gnupg2 lsb-release software-properties-common \
  build-essential cmake git python3-pip python3-venv
```

## Install ROS2 Humble
```
sudo apt install -y software-properties-common
sudo add-apt-repository -y universe
```

## Setup the ROS repository
```
sudo apt update
sudo apt install -y curl
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg
  
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" \
  | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

## Install ROS2 Humble Desktop
```
sudo apt update
sudo apt install -y ros-humble-desktop ros-dev-tools
```

## Add official Gazebo package repository
```
sudo curl https://packages.osrfoundation.org/gazebo.gpg \
  --output /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg
  
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] https://packages.osrfoundation.org/gazebo/ubuntu-stable jammy main" \
  | sudo tee /etc/apt/sources.list.d/gazebo-stable.list
  
sudo apt update
```

## Install Gazebo Harmonic and ROS integration
```
sudo apt install -y \
  gz-harmonic \
  ros-humble-ros-gzharmonic \
  ros-humble-ros-gzharmonic-bridge

# verify
gz sim --versions
ros2 pkg list | grep -E 'ros_gz_(sim|bridge)'
```


## Install Ardupilot build dependencies
```
sudo apt install -y \
  git git-lfs \
  python3-dev python3-opencv python3-pip python3-venv \
  python3-wxgtk4.0 python3-matplotlib python3-lxml \
  python3-pygame python3-yaml python3-setuptools \
  python3-serial python3-pexpect \
  g++ make ccache \
  libtool libxml2-dev libxslt1-dev \
  pkg-config \
  libffi-dev libssl-dev \
  libopencv-dev
```

## Install and run Ardupilot's prerequisite setup script

```
cd ~
git clone https://github.com/ArduPilot/ardupilot.git
cd ~/ardupilot
git submodule update --init --recursive
Tools/environment_install/install-prereqs-ubuntu.sh -y

# verify
git status --short
git rev-parse --short HEAD
git submodule status | head -20
```

## Clone Gazebo integration
```
cd ~
git clone https://github.com/ArduPilot/ardupilot_gz.git

# inspect
cd ~/ardupilot_gz
ls
find . -maxdepth 2 -name package.xml -print
```

## Create workspace and import dependencies
```
sudo apt install -y python3-vcstool python3-rosdep

mkdir -p ~/ardu_ws/src
cd ~/ardu_ws

vcs import --input ~/ardupilot_gz/ros2_gz.repos --recursive src
```

## Configure Gazebo Harmonic dependencies
```
echo 'export GZ_VERSION=harmonic' >> ~/.bashrc
source ~/.bashrc

sudo wget https://raw.githubusercontent.com/osrf/osrf-rosdep/master/gz/00-gazebo.list \
  -O /etc/ros/rosdep/sources.list.d/00-gazebo.list

sudo rosdep init 2>/dev/null || true
rosdep update
```

## Install workspace dependencies
```
cd ~/ardu_ws
source /opt/ros/humble/setup.bash
rosdep install --from-paths src --ignore-src -r -y

# verify
cd ~/ardu_ws
find src -maxdepth 2 -name package.xml -print
```

## Add gps_umd
```
cd ~/ardu_ws/src
git clone -b ros2-devel https://github.com/swri-robotics/gps_umd.git
```

## Configure the environment
```
grep -q 'export GZ_VERSION=harmonic' ~/.bashrc || \
  echo 'export GZ_VERSION=harmonic' >> ~/.bashrc

source ~/.bashrc
source /opt/ros/humble/setup.bash
```

## Install dependencies
```
cd ~/ardu_ws
sudo rosdep init 2>/dev/null || true
rosdep update
rosdep install --from-paths src --ignore-src -r -y

# verify
cd ~/ardu_ws
colcon list
```

## Exclude gpsd_client
```
touch ~/ardu_ws/src/gps_umd/gpsd_client/COLCON_IGNORE
```

## Install & Build & Add to path Micro-XRCE-DDS-Gen
```
sudo apt install -y default-jre
cd ~
git clone --recurse-submodules --branch v4.7.0 \
  https://github.com/ardupilot/Micro-XRCE-DDS-Gen.git
  
cd ~/Micro-XRCE-DDS-Gen
./gradlew assemble

grep -qxF 'export PATH="$PATH:$HOME/Micro-XRCE-DDS-Gen/scripts"' ~/.bashrc \
  || echo 'export PATH="$PATH:$HOME/Micro-XRCE-DDS-Gen/scripts"' >> ~/.bashrc

source ~/.bashrc

# verify
microxrceddsgen -help
```

## Patches
```
# robot.launch.py

python3 - <<'PY'
from pathlib import Path

p = Path.home() / "ardu_ws/src/ardupilot_gz/ardupilot_gz_bringup/launch/robots/robot.launch.py"
s = p.read_text()

old = '''    # Resolve package:// URIs for SDFormat nested-model includes.
    ardupilot_gazebo_models = os.path.join(
        get_package_share_directory("ardupilot_gazebo"), "models"
    )

    robot_desc = robot_desc.replace(
        "package://ardupilot_gazebo/models/",
        ardupilot_gazebo_models + "/",
    )
'''

if old in s:
    print("Patch already present:", p)
elif "Resolve package:// URIs for SDFormat nested-model includes." in s:
    raise SystemExit("Patch comment exists but content differs; not modifying.")
else:
    anchor = "    # Replace FDM port in robot description"
    if anchor not in s:
        raise SystemExit("Insertion point not found. File unchanged.")
    s = s.replace(anchor, old + "\n" + anchor, 1)
    p.write_text(s)
    print("Patched:", p)
PY

# iris.launch.py

python3 - <<'PY'
from pathlib import Path

p = Path.home() / "ardu_ws/src/ardupilot_gz/ardupilot_gz_bringup/launch/robots/iris.launch.py"
s = p.read_text()

patch = '''    # Resolve package:// mesh URIs for Gazebo.
    robot_desc = robot_desc.replace(
        "package://ardupilot_gazebo/models/",
        get_package_share_directory("ardupilot_gazebo") + "/models/",
    )
'''

if "Resolve package:// mesh URIs for Gazebo." in s:
    print("Patch already present:", p)
else:
    anchor = "    robot_desc = "
    # Insert immediately after the first robot_desc assignment that reads the model.
    lines = s.splitlines(keepends=True)
    for i, line in enumerate(lines):
        if anchor in line and "read_text" in line:
            lines.insert(i + 1, "\n" + patch)
            p.write_text("".join(lines))
            print("Patched:", p)
            break
    else:
        raise SystemExit("Insertion point not found. File unchanged.")
PY


# model patch

python3 - <<'PY'
from pathlib import Path

root = Path.home() / "ardu_ws/src/ardupilot_gazebo/models"

files = [
    root / "iris_with_standoffs/model.sdf",
    root / "gimbal_small_3d/model.sdf",
    root / "iris_with_gimbal/model.sdf",
]

old = "package://ardupilot_gazebo/models/"
new = "model://"

for p in files:
    s = p.read_text()
    count = s.count(old)
    s = s.replace(old, new)
    p.write_text(s)
    print(f"{p}: replaced {count} occurrence(s)")
PY
```

## Add MAVProxy PATH fix
```
grep -qxF 'export PATH="$HOME/.local/bin:$PATH"' ~/.bashrc ||
  echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc

source ~/.bashrc
```

## Add downward facing camera patch to `model.sdf`
```
python3 - <<'PY'
from pathlib import Path
import shutil

p = Path.home() / "ardu_ws/src/ardupilot_gazebo/models/iris_with_gimbal/model.sdf"
s = p.read_text()

if 'name="downward_camera_link"' in s:
    print("Downward camera already exists; no changes made.")
    raise SystemExit(0)

if s.count("</model>") != 1:
    raise SystemExit("Expected exactly one </model> tag; file unchanged.")

backup = p.with_suffix(".model.sdf.pre_downward_camera.bak")
shutil.copy2(p, backup)

patch = '''
    <!-- Downward-facing camera for ArUco precision landing -->
    <link name="downward_camera_link">
      <pose>0 0 -0.08 0 0 0</pose>
      <sensor name="downward_camera" type="camera">
        <pose>0 0 0 0 1.57079632679 0</pose>
        <always_on>true</always_on>
        <update_rate>30</update_rate>
        <visualize>true</visualize>
        <topic>/downward_camera/image</topic>
        <camera>
          <horizontal_fov>2.70</horizontal_fov>
          <image>
            <width>1280</width>
            <height>800</height>
          </image>
          <clip>
            <near>0.05</near>
            <far>100</far>
          </clip>
        </camera>
      </sensor>
    </link>

    <joint name="downward_camera_joint" type="fixed">
      <parent>base_link</parent>
      <child>downward_camera_link</child>
    </joint>
'''

p.write_text(s.replace("</model>", patch + "\n  </model>", 1))
print(f"Patched: {p}")
print(f"Backup:  {backup}")
PY
```

## Apply ROS bridge patch to `iris_bridge.yaml`
```
python3 - <<'PY'
from pathlib import Path
import shutil

p = Path.home() / "ardu_ws/src/ardupilot_gz/ardupilot_gz_bringup/config/iris_bridge.yaml"
s = p.read_text()

marker = '- ros_topic_name: "downward_camera/image"'
if marker in s:
    print("Downward-camera bridge already exists; no changes made.")
    raise SystemExit(0)

anchor = '''- ros_topic_name: "camera/camera_info"
  gz_topic_name: "/world/{{ world_name }}/model/{{ robot_name }}/link/pitch_link/sensor/camera/camera_info"
  ros_type_name: "sensor_msgs/msg/CameraInfo"
  gz_type_name: "gz.msgs.CameraInfo"
  direction: GZ_TO_ROS
'''

if anchor not in s:
    raise SystemExit("Existing camera mapping not found; file unchanged.")

backup = p.with_suffix(".yaml.bak")
shutil.copy2(p, backup)

patch = anchor + '''
- ros_topic_name: "downward_camera/image"
  gz_topic_name: "/downward_camera/image"
  ros_type_name: "sensor_msgs/msg/Image"
  gz_type_name: "gz.msgs.Image"
  direction: GZ_TO_ROS
- ros_topic_name: "downward_camera/camera_info"
  gz_topic_name: "/downward_camera/camera_info"
  ros_type_name: "sensor_msgs/msg/CameraInfo"
  gz_type_name: "gz.msgs.CameraInfo"
  direction: GZ_TO_ROS
'''

p.write_text(s.replace(anchor, patch, 1))
print(f"Patched: {p}")
print(f"Backup:  {backup}")
PY
```

## Generate ArUco markers
```

mkdir -p ~/ardu_ws/src/ardupilot_gazebo/tags
python3 - <<'PY'
from pathlib import Path
import cv2

root = Path.home() / "ardu_ws/src/ardupilot_gazebo/tags"
aruco = cv2.aruco
dictionary = aruco.getPredefinedDictionary(aruco.DICT_ARUCO_ORIGINAL)

for marker_id in (72, 10):
    d = root / f"aruco_marker_{marker_id}"
    d.mkdir(parents=True, exist_ok=True)

    image = aruco.generateImageMarker(
        dictionary, marker_id, 600, borderBits=1
    )
    cv2.imwrite(str(d / f"aruco_{marker_id}.png"), image)

    (d / "model.sdf").write_text(f'''<?xml version="1.0"?>
<sdf version="1.7">
  <model name="aruco_marker_{marker_id}">
    <static>true</static>
    <link name="link">
      <visual name="visual">
        <cast_shadows>false</cast_shadows>
        <geometry>
          <box><size>0.15 0.15 0.001</size></box>
        </geometry>
        <material>
          <ambient>1 1 1 1</ambient>
          <diffuse>1 1 1 1</diffuse>
          <pbr>
            <metal>
              <albedo_map>aruco_{marker_id}.png</albedo_map>
              <metalness>0</metalness>
              <roughness>1</roughness>
            </metal>
          </pbr>
        </material>
      </visual>
    </link>
  </model>
</sdf>
''')

    (d / "model.config").write_text(f'''<?xml version="1.0"?>
<model>
  <name>aruco_marker_{marker_id}</name>
  <version>1.0</version>
  <sdf version="1.7">model.sdf</sdf>
</model>
''')

    print(f"Created marker {marker_id}: {d}")
PY

```

## Place markers in the world
```
python3 - <<'PY'
from pathlib import Path
import xml.etree.ElementTree as ET

world_path = (
    Path.home()
    / "ardu_ws/src/ardupilot_gz/ardupilot_gz_gazebo/worlds/runway.sdf"
)
tags = Path.home() / "ardu_ws/src/ardupilot_gazebo/tags"

tree = ET.parse(world_path)
root = tree.getroot()
world = root.find("world")
if world is None:
    raise RuntimeError("No <world> element found")

poses = {
    "aruco_marker_72": "-0.125 0 0.001 0 0 0",
    "aruco_marker_10": "0.125 0 0.001 0 0 0",
}

for name, pose_text in poses.items():
    entries = [
        e for e in world.findall("include")
        if e.findtext("name") == name
    ]

    if entries:
        entry = entries[0]
        for duplicate in entries[1:]:
            world.remove(duplicate)
    else:
        entry = ET.SubElement(world, "include")
        ET.SubElement(entry, "uri").text = f"file://{tags / name}"

    ET.SubElement(entry, "name") if entry.find("name") is None else None
    entry.find("name").text = name

    pose = entry.find("pose")
    if pose is None:
        pose = ET.SubElement(entry, "pose")
    pose.text = pose_text

ET.indent(tree, space="  ")
tree.write(world_path, encoding="UTF-8", xml_declaration=True)
print(f"Updated: {world_path}")
print("ID 72: x=-0.125 m; ID 10: x=+0.125 m")
PY

```

## Patch for adding mass and inertia to camera
```
python3 - <<'PY'
from pathlib import Path
import xml.etree.ElementTree as ET

path = Path.home() / "ardu_ws/src/ardupilot_gazebo/models/iris_with_gimbal/model.sdf"
tree = ET.parse(path)
root = tree.getroot()

link = root.find(".//link[@name='downward_camera_link']")
if link is None:
    raise SystemExit("downward_camera_link not found")

if link.find("inertial") is not None:
    raise SystemExit("An inertial block already exists; not modifying it.")

inertial = ET.Element("inertial")
ET.SubElement(inertial, "mass").text = "0.05"

inertia = ET.SubElement(inertial, "inertia")
# Solid-box approximation: 0.03 x 0.03 x 0.02 m, mass 0.05 kg
ET.SubElement(inertia, "ixx").text = "4.1667e-6"
ET.SubElement(inertia, "iyy").text = "4.1667e-6"
ET.SubElement(inertia, "izz").text = "7.5e-6"
ET.SubElement(inertia, "ixy").text = "0"
ET.SubElement(inertia, "ixz").text = "0"
ET.SubElement(inertia, "iyz").text = "0"

link.insert(0, inertial)
ET.indent(tree, space="  ")
tree.write(path, encoding="UTF-8", xml_declaration=True)
print("Added 50 g camera mass and inertia.")
PY
```
## Build the workspace
```
cd ~/ardu_ws
source /opt/ros/humble/setup.bash
export GZ_VERSION=harmonic

rosdep install --from-paths src --ignore-src -r -y

colcon build --symlink-install
```

## Launch Simulation
```
cd ~/ardu_ws
source /opt/ros/humble/setup.bash
source install/setup.bash
export GZ_VERSION=harmonic

ros2 launch ardupilot_gz_bringup iris_runway.launch.py \
  rviz:=true use_gz_tf:=true
```


## Patch to modify wind speed
```
python3 - <<'PY'
from pathlib import Path

p = Path.home() / "ardu_ws/src/ardupilot_gz/ardupilot_gz_gazebo/worlds/runway.sdf"
s = p.read_text()

old = "<linear_velocity>5 0 0</linear_velocity>"
new = "<linear_velocity>8 0 0</linear_velocity>"

if s.count(old) != 1:
    raise SystemExit("Expected wind setting not found exactly once; no change made.")

p.write_text(s.replace(old, new, 1))
print("Wind velocity changed to 8 m/s along world X.")
PY
```

