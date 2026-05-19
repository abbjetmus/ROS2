# Labb 5: Publisher-nod i Python

**Skriv en egen nod som styr sköldpaddan**

**Förkunskaper:** Labb 1–4

## Syfte

I labb 3 skickade du `cmd_vel`-meddelanden manuellt från terminalen. Nu skriver du en Python-nod som gör samma sak automatiskt — en **publisher**. Det är samma princip som används för att styra riktiga robotar.

## Mål

Efter labben ska du kunna:

- Skapa en publisher med `create_publisher`.
- Använda `create_timer` för att skicka meddelanden regelbundet.
- Bygga ett `Twist`-meddelande i Python och publicera det.
- Förändra publicerade värden i realtid baserat på tid eller logik.

## Del 1: Repetera workspacet

Om du tappat bort `min_turtle` från labb 4, skapa den igen enligt instruktionerna där. Aktivera workspacet:

```bash
cd ~/ros2_ws
source install/setup.bash
```

Starta turtlesim i en separat terminal:

```bash
ros2 run turtlesim turtlesim_node
```

## Del 2: Skelett för publisher

Skapa filen `src/min_turtle/min_turtle/cirkel.py`:

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist


class CirkelNod(Node):
    def __init__(self):
        super().__init__('cirkel_nod')
        self.publisher = self.create_publisher(Twist, '/turtle1/cmd_vel', 10)
        self.timer = self.create_timer(0.1, self.skicka_kommando)
        self.get_logger().info('Cirkel-nod startad')

    def skicka_kommando(self):
        msg = Twist()
        msg.linear.x = 2.0
        msg.angular.z = 1.0
        self.publisher.publish(msg)


def main():
    rclpy.init()
    node = CirkelNod()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Förklaring rad-för-rad:

| Rad | Vad den gör |
|---|---|
| `create_publisher(Twist, '/turtle1/cmd_vel', 10)` | Skapar en publisher på topicet, med kö-storlek 10. |
| `create_timer(0.1, self.skicka_kommando)` | Anropa metoden var 0.1 sekund (10 Hz). |
| `msg = Twist()` | Skapa ett tomt Twist-meddelande. |
| `publisher.publish(msg)` | Skicka meddelandet. |
| `rclpy.spin(node)` | Kör noden tills den stoppas med Ctrl+C. |

## Del 3: Registrera och bygg

Lägg till i `setup.py` under `console_scripts`:

```python
'cirkel = min_turtle.cirkel:main',
```

Bygg och kör:

```bash
cd ~/ros2_ws
colcon build --packages-select min_turtle
source install/setup.bash
ros2 run min_turtle cirkel
```

Sköldpaddan ska börja köra i cirkel.

## Uppgifter

### Uppgift 1 — Stora och små cirklar

Ändra `linear.x` och `angular.z` så att sköldpaddan ritar:

- **a)** En cirkel med radie ungefär 1 enhet.
- **b)** En cirkel med radie ungefär 3 enheter.
- **c)** En motsatt riktad cirkel (medurs istället för moturs).

Förklara med en mening hur radien beror på de två värdena.

### Uppgift 2 — Spiral

Modifiera noden så att sköldpaddan rör sig i en **spiral** som öppnar sig utåt. Tips: låt `linear.x` öka långsamt över tid medan `angular.z` är konstant.

```python
self.t = 0.0

def skicka_kommando(self):
    self.t += 0.1
    msg = Twist()
    msg.linear.x = ...     # öka med tiden
    msg.angular.z = ...    # konstant
    self.publisher.publish(msg)
```

Skärmdump av spiralen.

### Uppgift 3 — Fyrkant

Skriv en ny nod `src/min_turtle/min_turtle/fyrkant.py` som får sköldpaddan att rita en fyrkant. Använd ett tillstånd som växlar mellan "kör rakt" och "sväng 90 grader".

Förslag på struktur:

```python
class FyrkantNod(Node):
    def __init__(self):
        super().__init__('fyrkant_nod')
        self.publisher = self.create_publisher(Twist, '/turtle1/cmd_vel', 10)
        self.timer = self.create_timer(0.1, self.tick)
        self.steg = 0          # vilket "steg" vi är i
        self.tid_i_steg = 0.0  # hur länge vi varit i steget

    def tick(self):
        msg = Twist()
        if self.steg % 2 == 0:
            # Kör rakt i 2 sekunder
            msg.linear.x = 1.0
            if self.tid_i_steg > 2.0:
                self.steg += 1
                self.tid_i_steg = 0.0
        else:
            # Sväng 90° (cirka pi/2 radianer)
            msg.angular.z = ...
            if self.tid_i_steg > ...:
                self.steg += 1
                self.tid_i_steg = 0.0

        self.publisher.publish(msg)
        self.tid_i_steg += 0.1
```

Fyll i värdena så att svängen blir 90°. Tips: om `angular.z = 1.0` rad/s, hur lång tid tar en kvarts vridning?

Skärmdump av fyrkanten.

### Uppgift 4 — Stjärna (utmaning)

Bygg vidare på `fyrkant.py` och skapa `stjarna.py` som ritar en femuddig stjärna. Tips: vinkeln mellan strecken i en femuddig stjärna är 144°.

### Uppgift 5 — Bokstaven i ditt namn (utmaning)

Skriv en nod som ritar första bokstaven i ditt namn utan teleop. Du får använda services (t.ex. `teleport_absolute` och `set_pen`) tillsammans med din publisher.

Tips för att anropa services från Python:

```python
from turtlesim.srv import TeleportAbsolute
self.tp_client = self.create_client(TeleportAbsolute, '/turtle1/teleport_absolute')
self.tp_client.wait_for_service()
req = TeleportAbsolute.Request()
req.x = 5.0
req.y = 5.0
req.theta = 0.0
self.tp_client.call_async(req)
```

## Inlämning

1. Värden från uppgift 1 + en mening om radien.
2. `spiral.py` (eller modifierad `cirkel.py`) + skärmdump.
3. `fyrkant.py` + skärmdump.
4. Antingen `stjarna.py` eller bokstavsnoden + skärmdump.

## Vanliga problem

**Sköldpaddan rör sig inte fast noden körs** — kör turtlesim igång? Kollar du rätt topic-namn (`/turtle1/cmd_vel`, inte `/cmd_vel`)?

**`ImportError: No module named 'min_turtle'`** — du körde inte `source install/setup.bash` i terminalen där du startar noden.

**Sköldpaddan flyger iväg när noden stoppas inte ordentligt** — om du startar samma nod två gånger får sköldpaddan dubbla kommandon. Avsluta gamla körningar med `Ctrl+C` först.
