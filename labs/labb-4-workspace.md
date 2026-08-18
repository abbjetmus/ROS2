# Labb 4: Workspace och package

**Skapa ett eget ROS2-workspace och ditt första package**

**Förkunskaper:** Labb 1–3

## Syfte

Du ska bygga den katalogstruktur som krävs för att skriva egna ROS2-noder. Efter labben har du ett fungerande **workspace** med ett tomt **Python-package** som du kommer fortsätta utveckla i labb 5–8.

## Mål

Efter labben ska du kunna:

- Förklara vad ett **workspace** och ett **package** är.
- Skapa ett package med `ros2 pkg create`.
- Bygga workspace med `colcon build`.
- Använda `source install/setup.bash` och förklara varför det behövs.

## Bakgrund

| Begrepp | Förklaring |
|---|---|
| **Workspace** | En mapp som innehåller alla dina ROS2-projekt. Hos oss: `~/ros2_ws`. |
| **Package** | Ett enskilt ROS2-projekt med noder, konfiguration och beroenden. Ligger i `~/ros2_ws/src/`. |
| **colcon** | Byggverktyget som kompilerar och installerar dina packages. |

## Del 0: Installera colcon

`colcon` ingår inte alltid i en minimal ROS2-installation. Installera det en gång:

```bash
sudo apt update
sudo apt install -y python3-colcon-common-extensions
```

## Del 1: Skapa workspace

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
```

Bygg det tomma workspacet:

```bash
colcon build
```

Efter byggningen finns tre nya mappar: `build/`, `install/` och `log/`. Efter varje bygge måste du köra:

```bash
source install/setup.bash
```

> Detta talar om för terminalen var dina nybyggda packages finns. Annars hittar inte `ros2 run` dem.

## Del 2: Skapa ditt första package

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python \
  --license Apache-2.0 \
  --dependencies rclpy geometry_msgs std_msgs turtlesim \
  min_turtle
```

Detta skapar en mapp `min_turtle/` med:

```text
min_turtle/
├── min_turtle/        ← här lägger du Python-filer
│   └── __init__.py
├── package.xml        ← metadata och beroenden
├── setup.py           ← beskriver hur package installeras
├── setup.cfg
├── resource/
└── test/
```

## Del 3: Skriv en trivial nod

Öppna VS Code i workspacet:

```bash
cd ~/ros2_ws
code .
```

Skapa filen `src/min_turtle/min_turtle/hej.py`:

```python
import rclpy
from rclpy.node import Node


class HejNod(Node):
    def __init__(self):
        super().__init__('hej_nod')
        self.get_logger().info('Hej från min första ROS2-nod!')


def main():
    rclpy.init()
    node = HejNod()
    rclpy.spin_once(node, timeout_sec=1.0)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

## Del 4: Registrera noden i `setup.py`

Öppna `src/min_turtle/setup.py`. Hitta `entry_points` och ändra till:

```python
entry_points={
    'console_scripts': [
        'hej = min_turtle.hej:main',
    ],
},
```

## Del 5: Bygg och kör

```bash
cd ~/ros2_ws
colcon build --packages-select min_turtle
source install/setup.bash
ros2 run min_turtle hej
```

Du ska se ungefär:

```text
[INFO] [hej_nod]: Hej från min första ROS2-nod!
```

## Koppling till robotmoppen

Robotmoppens mjukvara samlas senare i ett eget package. Där kan gruppen ha exempelvis:

```text
robotmopp_system/
├── controller_node.py
├── sensor_node.py
├── safety_node.py
├── motor_node.py
└── launch/
```

Workspacet gör att alla delar kan byggas och startas på ett ordnat sätt.

**Kopplingsfråga:** Vad är skillnaden mellan workspacet och robotmoppens package?

Svara med 1–2 meningar. Svaret ingår i inlämningen.

## Inlämning

1. Skärmdumpar från uppgift 1, 2 och 4.
2. Förklaring från uppgift 3.
3. En kort beskrivning av skillnaden mellan workspace och package.
4. Kort svar på kopplingsfrågan om robotmoppen.

Uppgift 5 är frivillig fördjupning.
