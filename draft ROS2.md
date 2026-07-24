# Simular un dron que “ve” usando ROS 2 Humble, aunque no tengas Pixhawk 6X, cámara real ni dron físico.
Todo será en simulación, usando:

- Cámara simulada en Gazebo
- ROS 2 image_transport
- Nodo de visión (placeholder primero, YOLOv8 después)
- Publicación de /objects_detected y /detection_pose
- Rviz2 para visualizar
- Mantener tu PX4 SITL intacto



Tengo un windows 11 education, WSL2, Ubuntu 22.04 jammy, PX4 SITL v1.15, Gazebo 6.18, ROS 2 Humble y QGroundControl. Mi idea es pilotar un dron, pero primero necesitaba simularlo (ya lo logré mediante gazebo, px4 y qgroundcontrol), ahora quiero aprender sobre ros 2 humble y yolov8. 
Se me solicita: Adoptar ROS 2 Humble como framework estándar.
Un comando lanta PX4 SITL + Gazebo + Rviz + nodo de visión orquestados por ROS 2:
- image_transport  
- Rviz 2 
- SLAM intro
- /objects_detected
Que el dron pase de ser ciego a ver el entorno y reaccionar:
- YOLOv8 a 15+FPS en Jetson
- modelo lab_uav_v1.pt
- detection_pose publicando al FC
- YOLOv8, custom training, DeepSORT/ByteTrack
























Vamos paso a paso, sin romper nada de lo que ya tienes. Voy a asumir:

- PX4 SITL y Gazebo ya funcionan con tus mundos.
- ROS 2 Humble está instalado en tu Ubuntu 22.04 (WSL2).

Si algo de esto no es cierto, me lo dices antes de ejecutar comandos.

---

### 1. Crear workspace ROS 2 Humble (sin tocar nada existente)

En tu Ubuntu (WSL2):

```bash
mkdir -p ~/ros2_uav_ws/src
cd ~/ros2_uav_ws
colcon build
source install/setup.bash
```

Para no tener que escribir esto cada vez, añade al final de `~/.bashrc`:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_uav_ws/install/setup.bash
```

Luego:

```bash
source ~/.bashrc
```

---

### 2. Crear paquete `uav_bringup` (solo launch y config)

Dentro del workspace:

```bash
cd ~/ros2_uav_ws/src
ros2 pkg create uav_bringup --build-type ament_python
```

Esto crea la estructura básica. Ahora:

```bash
cd uav_bringup
mkdir launch rviz
```

Edita `setup.py` para que quede algo así (solo lo importante):

```python
from setuptools import setup

package_name = 'uav_bringup'

setup(
    name=package_name,
    version='0.0.0',
    packages=[package_name],
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        ('share/' + package_name + '/launch', ['launch/uav_full_sim.launch.py']),
        ('share/' + package_name + '/rviz', ['rviz/uav.rviz']),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='tu_nombre',
    maintainer_email='tu_email@example.com',
    description='Bringup PX4 + Gazebo + vision + Rviz2',
    license='Apache-2.0',
    tests_require=['pytest'],
    entry_points={
        'console_scripts': [
        ],
    },
)
```

> No te preocupes por `uav.rviz` y `uav_full_sim.launch.py` todavía, los vamos a crear ahora.

---

### 3. Añadir cámara simulada en Gazebo (concepto)

Aquí hay dos variantes:

- Ya tienes un modelo de dron con cámara en tu mundo.
- No tienes cámara aún.

Para no romper nada, vamos a **usar un mundo nuevo de prueba** con una cámara simple, sin tocar tus mundos actuales.

Crea un mundo de prueba, por ejemplo:

```bash
mkdir -p ~/gazebo_worlds
nano ~/gazebo_worlds/uav_cam.world
```

Dentro, pon algo muy básico (ejemplo simplificado):

```xml
<?xml version="1.0" ?>
<sdf version="1.6">
  <world name="uav_cam_world">
    <include>
      <uri>model://sun</uri>
    </include>

    <model name="camera_model">
      <static>true</static>
      <link name="link">
        <pose>0 0 1 0 0 0</pose>
        <sensor name="camera" type="camera">
          <always_on>1</always_on>
          <update_rate>30</update_rate>
          <camera>
            <horizontal_fov>1.3962634</horizontal_fov>
            <image>
              <width>640</width>
              <height>480</height>
              <format>R8G8B8</format>
            </image>
            <clip>
              <near>0.1</near>
              <far>100</far>
            </clip>
          </camera>
        </sensor>
      </link>
    </model>
  </world>
</sdf>
```

Esto no toca tu dron ni PX4; es solo un mundo de prueba con una cámara.

---

### 4. Bridge de imagen Gazebo → ROS 2

Vamos a asumir que usas `ros_gz_bridge` (antes `ros_ign_bridge`) o similar. Si no lo tienes:

```bash
sudo apt install ros-humble-ros-gz-image ros-humble-ros-gz-bridge
```

El bridge típico de imagen es algo como:

```bash
ros2 run ros_gz_image image_bridge
```

Pero lo vamos a orquestar desde el launch.

---

### 5. Crear launch unificado `uav_full_sim.launch.py`

En `~/ros2_uav_ws/src/uav_bringup/launch`:

```bash
nano uav_full_sim.launch.py
```

Contenido (ajustaremos rutas luego):

```python
from launch import LaunchDescription
from launch.actions import ExecuteProcess
from launch_ros.actions import Node

def generate_launch_description():
    ld = LaunchDescription()

    # PX4 SITL (usa tu ruta real de PX4)
    px4 = ExecuteProcess(
        cmd=[
            '/home/TU_USUARIO/PX4-Autopilot/build/px4_sitl_default/bin/px4',
            '/home/TU_USUARIO/PX4-Autopilot/ROMFS/px4fmu_common',
            '-s', 'etc/init.d-posix/rcS'
        ],
        output='screen'
    )
    ld.add_action(px4)

    # Gazebo con mundo de prueba con cámara
    gazebo = ExecuteProcess(
        cmd=[
            'gazebo',
            '/home/TU_USUARIO/gazebo_worlds/uav_cam.world'
        ],
        output='screen'
    )
    ld.add_action(gazebo)

    # Bridge de imagen (ajusta si tu topic de Gazebo es distinto)
    camera_bridge = Node(
        package='ros_gz_image',
        executable='image_bridge',
        name='camera_bridge',
        output='screen',
        parameters=[{
            'use_sim_time': True
        }],
        remappings=[
            ('/camera', '/camera/image_raw')
        ]
    )
    ld.add_action(camera_bridge)

    # Nodo de visión dummy (lo crearemos luego)
    vision_node = Node(
        package='uav_vision',
        executable='vision_dummy',
        name='vision_dummy',
        output='screen',
        parameters=[{
            'input_image_topic': '/camera/image_raw',
            'objects_topic': '/objects_detected',
            'pose_topic': '/detection_pose'
        }]
    )
    ld.add_action(vision_node)

    # Rviz2
    rviz = Node(
        package='rviz2',
        executable='rviz2',
        name='rviz2',
        output='screen',
        arguments=['-d', '/home/TU_USUARIO/ros2_uav_ws/src/uav_bringup/rviz/uav.rviz']
    )
    ld.add_action(rviz)

    return ld
```

**Importante:**  
- Cambia `TU_USUARIO` por tu usuario real.  
- Si tu PX4 SITL se lanza con otro comando (por ejemplo `make px4_sitl gazebo`), podemos adaptarlo.

---

### 6. Crear paquete `uav_vision` con nodo dummy

Ahora sí, vamos a crear el paquete de visión, pero primero con un nodo “falso” que no usa YOLOv8, solo para probar el flujo.

```bash
cd ~/ros2_uav_ws/src
ros2 pkg create uav_vision --build-type ament_python
cd uav_vision
mkdir uav_vision
touch uav_vision/__init__.py
```

Edita `setup.py`:

```python
from setuptools import setup

package_name = 'uav_vision'

setup(
    name=package_name,
    version='0.0.0',
    packages=[package_name],
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='tu_nombre',
    maintainer_email='tu_email@example.com',
    description='Vision nodes for UAV',
    license='Apache-2.0',
    tests_require=['pytest'],
    entry_points={
        'console_scripts': [
            'vision_dummy = uav_vision.vision_dummy:main',
        ],
    },
)
```

Crea el nodo dummy:

```bash
nano uav_vision/vision_dummy.py
```

Contenido:

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Image
from std_msgs.msg import String

class VisionDummyNode(Node):
    def __init__(self):
        super().__init__('vision_dummy')

        self.declare_parameter('input_image_topic', '/camera/image_raw')
        self.declare_parameter('objects_topic', '/objects_detected')
        self.declare_parameter('pose_topic', '/detection_pose')

        input_image_topic = self.get_parameter('input_image_topic').get_parameter_value().string_value
        objects_topic = self.get_parameter('objects_topic').get_parameter_value().string_value
        pose_topic = self.get_parameter('pose_topic').get_parameter_value().string_value

        self.subscription = self.create_subscription(
            Image,
            input_image_topic,
            self.image_callback,
            10
        )

        self.objects_pub = self.create_publisher(String, objects_topic, 10)
        self.pose_pub = self.create_publisher(String, pose_topic, 10)

        self.get_logger().info(f"VisionDummyNode escuchando en {input_image_topic}")
        self.get_logger().info(f"Publicando objetos en {objects_topic}")
        self.get_logger().info(f"Publicando pose en {pose_topic}")

    def image_callback(self, msg: Image):
        # Aquí solo simulamos detecciones
        objects_msg = String()
        objects_msg.data = "dummy_object_detected"
        self.objects_pub.publish(objects_msg)

        pose_msg = String()
        pose_msg.data = "dummy_pose"
        self.pose_pub.publish(pose_msg)

def main(args=None):
    rclpy.init(args=args)
    node = VisionDummyNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

### 7. Crear config de Rviz2 básica

En `~/ros2_uav_ws/src/uav_bringup/rviz`:

```bash
nano uav.rviz
```

Pon algo mínimo, por ejemplo:

```yaml
Panels:
  - Class: rviz_common/Displays
    Name: Displays
Visualization Manager:
  Class: rviz_common/VisualizationManager
  Displays:
    - Class: rviz_default_plugins/Image
      Name: Camera
      Topic: /camera/image_raw
    - Class: rviz_default_plugins/TF
      Name: TF
    - Class: rviz_default_plugins/Marker
      Name: Objects
  Enabled: true
```

No es perfecto, pero sirve para empezar.

---

### 8. Compilar todo y probar el comando único

Desde el workspace:

```bash
cd ~/ros2_uav_ws
colcon build
source install/setup.bash
```

Si no hay errores, prueba:

```bash
ros2 launch uav_bringup uav_full_sim.launch.py
```

Deberías ver:

- PX4 SITL arrancando (aunque no esté conectado al mundo de prueba, por ahora).  
- Gazebo con el mundo `uav_cam.world` y la cámara.  
- El bridge de imagen.  
- El nodo `vision_dummy` publicando mensajes en `/objects_detected` y `/detection_pose`.  
- Rviz2 mostrando la imagen de la cámara.

---

### 9. Siguiente paso: conectar esto a tu dron simulado real

Cuando esto funcione, el siguiente paso será:

- Usar tu mundo real de PX4 + Gazebo (dron) en lugar de `uav_cam.world`.  
- Conectar la cámara del dron simulado al bridge ROS 2.  
- Reemplazar `vision_dummy` por un nodo YOLOv8 real.

Pero primero quiero que llegues a que **el comando único funciona**.

---




















































Cheryl, **el error es exactamente lo que parece**:

```
error: can't copy 'launch/uav_full_sim.launch.py': doesn't exist
```

Tu `setup.py` **está bien**, pero ROS 2 intenta copiar un archivo que **todavía no existe**:

```
uav_bringup/launch/uav_full_sim.launch.py
```

Y también intenta copiar:

```
uav_bringup/rviz/uav.rviz
```

Si esos dos archivos no existen, **colcon build falla**.

Vamos a solucionarlo **sin tocar nada más**, con pasos exactos y cero ambigüedad.

---

# ✅ SOLUCIÓN — Crear los archivos vacíos que faltan

### 1. Crear el launch vacío

```bash
cd ~/ros2_uav_ws/src/uav_bringup/launch
nano uav_full_sim.launch.py
```

Pega esto dentro (mínimo válido):

```python
from launch import LaunchDescription

def generate_launch_description():
    return LaunchDescription([])
```

Guarda:

- Ctrl+O  
- Enter  
- Ctrl+X  

---

### 2. Crear el archivo RViz vacío

```bash
cd ~/ros2_uav_ws/src/uav_bringup/rviz
nano uav.rviz
```

Contenido mínimo:

```
# RViz config placeholder
```

Guarda y cierra.

---

# ✅ 3. Volver a compilar

```bash
cd ~/ros2_uav_ws
colcon build
source install/setup.bash
```

Ahora **sí** debe compilar sin errores.

---

