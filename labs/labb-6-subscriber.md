# Labb 6: Subscriber-nod i Python

**Läs `/turtle1/pose` och reagera på vad du ser**

**Förkunskaper:** Labb 1–5

## Syfte

Hittills har dina noder bara skickat data. Nu ska du **läsa** data — du skriver en **subscriber** som lyssnar på sköldpaddans position och reagerar på den. Det är så riktiga robotar fungerar: sensorvärden in, beslut, kommandon ut.

## Mål

Efter labben ska du kunna:

- Skapa en subscriber med `create_subscription`.
- Tolka ett `turtlesim/msg/Pose`-meddelande.
- Kombinera en subscriber och en publisher i samma nod.
- Skriva ett villkor som ändrar robotens beteende baserat på sensorvärden.

## Del 1: En ren subscriber

Skapa filen `src/min_turtle/min_turtle/lyssnare.py`:

```python
import rclpy
from rclpy.node import Node
from turtlesim.msg import Pose


class LyssnareNod(Node):
    def __init__(self):
        super().__init__('lyssnare_nod')
        self.subscription = self.create_subscription(
            Pose, '/turtle1/pose', self.pose_callback, 10
        )
        self.get_logger().info('Lyssnar på /turtle1/pose')

    def pose_callback(self, msg: Pose):
        self.get_logger().info(
            f'x={msg.x:.2f}  y={msg.y:.2f}  theta={msg.theta:.2f}'
        )


def main():
    rclpy.init()
    node = LyssnareNod()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Registrera i `setup.py`:

```python
'lyssnare = min_turtle.lyssnare:main',
```

Bygg, kör och styr sköldpaddan med teleop i en annan terminal:

```bash
colcon build --packages-select min_turtle
source install/setup.bash
ros2 run min_turtle lyssnare
```

## Del 2: Förstå callbacks

Subscribern fungerar inte som en loop. ROS2 anropar din `pose_callback`-funktion **varje gång ett nytt meddelande kommer**. Om turtlesim publicerar `/turtle1/pose` på 60 Hz blir din callback anropad 60 ggr/s.

## Del 3: Kombinera publisher och subscriber

Skapa `src/min_turtle/min_turtle/stoppare.py`. Noden ska köra sköldpaddan framåt **tills den närmar sig kanten**, då stannar den.

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from turtlesim.msg import Pose


class Stoppare(Node):
    def __init__(self):
        super().__init__('stoppare')
        self.publisher = self.create_publisher(Twist, '/turtle1/cmd_vel', 10)
        self.subscription = self.create_subscription(
            Pose, '/turtle1/pose', self.pose_callback, 10
        )

    def pose_callback(self, msg: Pose):
        cmd = Twist()
        nara_kant = msg.x < 1.0 or msg.x > 10.0 or msg.y < 1.0 or msg.y > 10.0
        if nara_kant:
            cmd.linear.x = 0.0
            self.get_logger().info('Stop! Nära kanten.')
        else:
            cmd.linear.x = 2.0
        self.publisher.publish(cmd)


def main():
    rclpy.init()
    rclpy.spin(Stoppare())
    rclpy.shutdown()
```

Registrera, bygg och kör.

## Uppgifter

### Uppgift 1 — Tysta loggen

Att logga varje meddelande gör terminalen ohanterlig. Modifiera `lyssnare.py` så att den bara loggar **var hundrade** anrop. Tips: räkna i en instansvariabel `self.n`.

### Uppgift 2 — Avstånd från mitten

Modifiera `lyssnare.py` så att den loggar avståndet från mitten av skärmen (`5.5, 5.5`):

```python
import math
dx = msg.x - 5.5
dy = msg.y - 5.5
avstand = math.sqrt(dx*dx + dy*dy)
```

Kör noden, styr sköldpaddan i en cirkel, och se hur avståndet varierar.

### Uppgift 3 — Smart vändare

Bygg vidare på `stoppare.py`. Istället för att bara stanna när sköldpaddan är nära kanten, ska den **vända 180°** och fortsätta. Tips:

- Ha ett tillstånd `self.vander = False`.
- När du upptäcker kant, sätt `self.vander = True` och starta en räknare.
- När noden är i vänd-läget, publicera `angular.z = 1.5` och `linear.x = 0`.
- Efter cirka 2 sekunder, sluta vända och kör framåt igen.

### Uppgift 4 — Färgrapport

Skapa en ny nod `farg_rapport.py` som lyssnar på `/turtle1/color_sensor` (`turtlesim/msg/Color`) och loggar färgen **bara när den ändras**.

Tips:

```python
from turtlesim.msg import Color
# ...
self.senaste = None

def cb(self, msg):
    nuvarande = (msg.r, msg.g, msg.b)
    if nuvarande != self.senaste:
        self.get_logger().info(f'Färg byttes till {nuvarande}')
        self.senaste = nuvarande
```

Styr sköldpaddan över egna penn-spår från tidigare labbar och se färgrapporten ändras.

### Uppgift 5 — Måljakt (utmaning)

Skriv en nod `jaga_mal.py`. Slumpa ut ett mål (`mal_x`, `mal_y`) i konstruktorn och låt noden:

1. Räkna ut riktning mot målet i `pose_callback`.
2. Jämföra riktningen med sköldpaddans `theta`.
3. Publicera `angular.z` proportionellt mot vinkelfelet.
4. Publicera `linear.x` proportionellt mot avståndet.
5. När sköldpaddan är inom 0.2 enheter från målet, logga `MÅL!` och stoppa.

Tips för vinkelräkning:

```python
import math
vinkel_till_mal = math.atan2(self.mal_y - msg.y, self.mal_x - msg.x)
vinkelfel = vinkel_till_mal - msg.theta
# normalisera till [-pi, pi]
vinkelfel = math.atan2(math.sin(vinkelfel), math.cos(vinkelfel))
```

## Inlämning

1. `lyssnare.py` med var-hundrade-loggning (uppgift 1) + skärmdump.
2. Avståndsversionen av lyssnaren (uppgift 2) — skärmdump med tre olika avstånd.
3. `stoppare.py` med vänd-logik (uppgift 3).
4. `farg_rapport.py` + skärmdump där färgen byts minst en gång.
5. `jaga_mal.py` + en kort förklaring (3–5 meningar) av hur den fungerar.

## Vanliga problem

**Callbacken anropas aldrig** — turtlesim måste vara igång. Du måste också ha kört `source install/setup.bash` i terminalen.

**Sköldpaddan rycker** — om både teleop och din nod publicerar `cmd_vel` samtidigt kommer sköldpaddan få blandade kommandon. Stäng teleop när du testar dina egna noder.

**`AttributeError: 'Pose' object has no attribute 'position'`** — `turtlesim/msg/Pose` har fälten `x`, `y`, `theta`, `linear_velocity`, `angular_velocity`. Det är **inte** samma som `geometry_msgs/Pose`.
