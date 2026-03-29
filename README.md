# ROS 2 Publisher & Subscriber Guide

A beginner's guide to creating a ROS 2 workspace, building a Python package, and writing publisher and subscriber nodes.

---

## 1. Create a workspace

A workspace is just a directory where all your ROS 2 packages live. The `src/` folder inside it is where your package source code goes.

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
```

---

## 2. Create a package

A ROS 2 package is the basic unit of code organisation. The `--build-type ament_python` flag tells ROS 2 this is a Python package. The `--dependencies` flag automatically adds `rclpy` (the ROS 2 Python client library) and `std_msgs` (standard message types) to your `package.xml`.

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python my_ros_pkg --dependencies rclpy std_msgs
```

This generates the following structure:

```
ros2_ws/
└── src/
    └── my_ros_pkg/
        ├── package.xml
        ├── setup.py
        ├── setup.cfg
        ├── resource/
        │   └── my_ros_pkg
        └── my_ros_pkg/
            └── __init__.py
```

Your node files go inside the inner `my_ros_pkg/` folder (the Python package), alongside `__init__.py`.

---

## 3. Create the nodes

### 3.1 Publisher node

Create the file `~/ros2_ws/src/my_ros_pkg/my_ros_pkg/publisher.py`:

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class Publisher(Node):
    def __init__(self):
        super().__init__('talker')
        self.pub = self.create_publisher(String, 'chatter', 10)
        self.timer = self.create_timer(0.1, self.timer_callback)  # 10 Hz
        self.counter = 0

    def timer_callback(self):
        msg = String()
        msg.data = f'Hello from ROS 2! Message count: {self.counter}'
        self.get_logger().info(msg.data)
        self.pub.publish(msg)
        self.counter += 1

def main(args=None):
    rclpy.init(args=args)
    node = Publisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

#### Line-by-line explanation

| Code | What it does |
|------|--------------|
| `import rclpy` | Imports the ROS 2 Python client library |
| `from rclpy.node import Node` | Imports the base `Node` class that all ROS 2 nodes inherit from |
| `from std_msgs.msg import String` | Imports the `String` message type from the `std_msgs` package |
| `class Publisher(Node)` | Defines the node as a Python class inheriting from `Node` |
| `super().__init__('talker')` | Calls the parent constructor and sets the node name to `talker` |
| `self.create_publisher(String, 'chatter', 10)` | Creates a publisher on the `chatter` topic using `String` messages, with a queue size of 10 |
| `self.create_timer(0.1, self.timer_callback)` | Calls `timer_callback` every 0.1 seconds (10 Hz) |
| `msg.data = f'Hello...'` | Sets the string content of the message |
| `self.get_logger().info(msg.data)` | Logs the message to the ROS 2 console |
| `self.pub.publish(msg)` | Sends the message out on the `chatter` topic |
| `rclpy.init(args=args)` | Initialises the ROS 2 Python runtime |
| `rclpy.spin(node)` | Keeps the node alive and processing callbacks |
| `node.destroy_node()` | Cleans up the node on shutdown |
| `rclpy.shutdown()` | Shuts down the ROS 2 runtime |

---

### 3.2 Subscriber node

Create the file `~/ros2_ws/src/my_ros_pkg/my_ros_pkg/subscriber.py`:

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class Subscriber(Node):
    def __init__(self):
        super().__init__('listener')
        self.sub = self.create_subscription(
            String,
            'chatter',
            self.callback,
            10
        )

    def callback(self, msg):
        self.get_logger().info(f'Received: {msg.data}')

def main(args=None):
    rclpy.init(args=args)
    node = Subscriber()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

#### Line-by-line explanation

| Code | What it does |
|------|--------------|
| `super().__init__('listener')` | Sets the node name to `listener` |
| `self.create_subscription(String, 'chatter', self.callback, 10)` | Subscribes to the `chatter` topic, calling `self.callback` each time a message arrives. Queue size is 10 |
| `def callback(self, msg)` | This function is called automatically whenever a new message is received |
| `msg.data` | Accesses the string content of the received message |

> **Note:** The topic name (`'chatter'`) and message type (`String`) must match exactly between the publisher and subscriber for communication to work.

---

## 4. Register nodes in `setup.py`

For `ros2 run` to find your nodes, they must be registered as **console scripts** in `setup.py`. Open `~/ros2_ws/src/my_ros_pkg/setup.py` and make sure it looks like this:

```python
from setuptools import find_packages, setup

package_name = 'my_ros_pkg'

setup(
    name=package_name,
    version='0.0.0',
    packages=find_packages(exclude=['test']),
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='your_name',
    maintainer_email='your@email.com',
    description='Simple ROS 2 publisher and subscriber',
    license='Apache-2.0',
    entry_points={
        'console_scripts': [
            'talker   = my_ros_pkg.publisher:main',
            'listener = my_ros_pkg.subscriber:main',
        ],
    },
)
```

The `entry_points` section is what matters here. The format is:

```
'<command_name> = <package>.<module>:<function>'
```

So `'talker = my_ros_pkg.publisher:main'` means: when someone runs `ros2 run my_ros_pkg talker`, execute the `main()` function inside `my_ros_pkg/publisher.py`.

> **Common mistake:** Using a dot instead of a colon before `main` (e.g. `my_ros_pkg.publisher.main`) will cause the node to not be found. Always use a colon: `my_ros_pkg.publisher:main`.

---

## 5. Build and run

### Build the package

```bash
cd ~/ros2_ws
colcon build --packages-select my_ros_pkg
```

### Source the workspace

This must be done in every new terminal before using your package. To avoid doing it manually every time, add it to your `~/.bashrc`:

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

Replace `humble` with your ROS 2 distribution if different (e.g. `iron`, `jazzy`).

### Run the nodes

Open two terminals. In each one, make sure the workspace is sourced.

**Terminal 1 — start the publisher:**
```bash
ros2 run my_ros_pkg talker
```

**Terminal 2 — start the subscriber:**
```bash
ros2 run my_ros_pkg listener
```

You should see output like this in the subscriber terminal:

```
[INFO] [listener]: Received: Hello from ROS 2! Message count: 0
[INFO] [listener]: Received: Hello from ROS 2! Message count: 1
[INFO] [listener]: Received: Hello from ROS 2! Message count: 2
```

> **Note:** Unlike ROS 1, ROS 2 does **not** require a `roscore` to be running. You can start nodes directly.

---

## 6. Useful debug commands

```bash
# List all active topics
ros2 topic list

# Print messages live on the chatter topic
ros2 topic echo /chatter

# Check the publish rate
ros2 topic hz /chatter

# See node info and connections
ros2 node info /talker
ros2 node info /listener
```
