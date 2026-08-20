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

`colcon` ingår inte i ROS2-installationen från labb 1. Installera det en gång:

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

Efter byggningen finns tre nya mappar: `build/`, `install/` och `log/`. Lägg på minnet — efter varje bygge måste du köra:

```bash
source install/setup.bash
```

> Detta talar om för terminalen var dina nybyggda packages finns. Annars hittar inte `ros2 run` dem.

## Del 2: Skapa ditt första package

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python \
  --license Apache-2.0 \
  --dependencies rclpy geometry_msgs turtlesim \
  min_turtle
```

Detta skapar en mapp `min_turtle/` med:

```
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

Öppna VS Code i workspacet om du inte redan gjort det:

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

Du ska se:

```
[INFO] [hej_nod]: Hej från min första ROS2-nod!
```

## Uppgifter

### Uppgift 1 — Förstå mappstrukturen

Kör `tree -L 3 ~/ros2_ws` (eller `ls -R` om `tree` inte finns). Skärmdump av utdata. Markera i bilden:

- Var ligger källkoden?
- Var ligger den **byggda** versionen som `ros2 run` använder?

### Uppgift 2 — Ditt namn

Ändra texten i `hej.py` så att den loggar `Hej, jag heter <ditt namn>!`. Bygg om och kör. Skärmdump av utdata.

### Uppgift 3 — Vad händer utan `source`?

Öppna en **ny terminal** och kör direkt:

```bash
ros2 run min_turtle hej
```

Vad händer? Kör sedan `source ~/ros2_ws/install/setup.bash` och försök igen. Förklara skillnaden.

### Uppgift 4 — Lägg till en kommandoradsparameter

Ändra `hej.py` så att noden tar emot ett namn som **ROS2-parameter** och loggar `Hej, <namn>!`:

```python
class HejNod(Node):
    def __init__(self):
        super().__init__('hej_nod')
        self.declare_parameter('namn', 'okänd')
        namn = self.get_parameter('namn').value
        self.get_logger().info(f'Hej, {namn}!')
```

Kör med:

```bash
ros2 run min_turtle hej --ros-args -p namn:=Sofia
```

Skärmdump där parametern syns i utdatat.

### Uppgift 5 — Andra noden (utmaning)

Skapa en ny fil `src/min_turtle/min_turtle/raknare.py` som loggar `Tick N` varje sekund (där N ökar från 1 och uppåt). Använd en `self.create_timer(1.0, self.tick)` i konstruktorn och låt `tick`-metoden öka en räknare.

Tips:

```python
self.n = 0
self.timer = self.create_timer(1.0, self.tick)

def tick(self):
    self.n += 1
    self.get_logger().info(f'Tick {self.n}')
```

Kom ihåg att i `main` byta `spin_once` till `spin(node)` så att timern fortsätter köra.

Registrera `raknare = min_turtle.raknare:main` i `setup.py`, bygg om, och kör med `ros2 run min_turtle raknare`.

## Inlämning

1. Skärmdumpar från uppgift 1, 2 och 4.
2. Förklaring från uppgift 3.
3. Filen `raknare.py` + skärmdump som visar minst fem tickar.
4. En kort beskrivning (3–5 meningar) av skillnaden mellan workspace och package.

## Användbara kommandon

```bash
ros2 pkg create --build-type ament_python --dependencies ... <namn>
colcon build                              # bygg allt
colcon build --packages-select <namn>     # bygg ett package
source install/setup.bash                 # aktivera bygget
ros2 run <pkg> <executable>               # kör en nod
ros2 pkg list                             # alla installerade packages
```
