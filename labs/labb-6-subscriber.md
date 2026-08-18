# Labb 6: Subscriber och sensordata

**Läs data från turtlesim och reagera med enkel logik**

**Förkunskaper:** Labb 1–5

## Syfte

I labb 5 skrev du en nod som **skickade** data. Nu ska du skriva en nod som **tar emot** data — en subscriber.

Du kommer läsa sköldpaddans position från `/turtle1/pose` och använda enkla villkor för att reagera på informationen.

Fokus är:

```text
sensordata in → enkel kontroll → kommando ut
```

Inte avancerad autonom navigation.

## Mål

Efter labben ska du kunna:

- Skapa en subscriber med `create_subscription`.
- Förklara vad en callback är.
- Läsa fält ur ett `turtlesim/msg/Pose`-meddelande.
- Kombinera en subscriber och en publisher i samma nod.
- Använda enkla `if`-villkor för att reagera på data.

## Del 1: En ren subscriber

Skapa filen:

```text
src/min_turtle/min_turtle/lyssnare.py
```

```python
import rclpy
from rclpy.node import Node
from turtlesim.msg import Pose


class LyssnareNod(Node):
    def __init__(self):
        super().__init__('lyssnare_nod')
        self.subscription = self.create_subscription(
            Pose,
            '/turtle1/pose',
            self.pose_callback,
            10
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

Bygg:

```bash
cd ~/ros2_ws
colcon build --packages-select min_turtle
source install/setup.bash
```

Starta sedan:

```bash
# Terminal 1
ros2 run turtlesim turtlesim_node

# Terminal 2
ros2 run turtlesim turtle_teleop_key

# Terminal 3
ros2 run min_turtle lyssnare
```

## Del 2: Vad är en callback?

Subscribern kör inte en vanlig `while`-loop.

När ett nytt meddelande kommer på `/turtle1/pose` anropar ROS2 automatiskt:

```python
self.pose_callback(msg)
```

`msg` innehåller bland annat:

```text
x
y
theta
linear_velocity
angular_velocity
```

Det här mönstret används hela tiden i robotik: avståndssensorer, knappar, batterivärden och statusmeddelanden kommer in via callbacks.

## Del 3: Kombinera subscriber och publisher

Nu bygger vi en mycket enkel säkerhetsfunktion: sköldpaddan kör framåt tills den är nära kanten. Då stannar den.

Skapa:

```text
src/min_turtle/min_turtle/stoppare.py
```

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from turtlesim.msg import Pose


class Stoppare(Node):
    def __init__(self):
        super().__init__('stoppare')

        self.publisher = self.create_publisher(
            Twist,
            '/turtle1/cmd_vel',
            10
        )

        self.subscription = self.create_subscription(
            Pose,
            '/turtle1/pose',
            self.pose_callback,
            10
        )

    def pose_callback(self, msg: Pose):
        cmd = Twist()

        nara_kant = (
            msg.x < 1.0
            or msg.x > 10.0
            or msg.y < 1.0
            or msg.y > 10.0
        )

        if nara_kant:
            cmd.linear.x = 0.0
        else:
            cmd.linear.x = 2.0

        self.publisher.publish(cmd)


def main():
    rclpy.init()
    node = Stoppare()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Registrera:

```python
'stoppare = min_turtle.stoppare:main',
```

Bygg och kör.

> Den här noden är medvetet enkel. Den ska lära dig kopplingen **subscriber → beslut → publisher**. Den ska inte lösa autonom navigation.

---

## Koppling till robotmoppen

På den fysiska robotmoppen kommer en sensornod läsa ett riktigt sensorvärde. Den kan sedan publicera:

```text
/robotmopp/safety_stop = true eller false
```

Säkerhetsnoden subscriberar på signalen och bestämmer om ett framåtkommando får skickas vidare. Principen är samma som i den här labben:

```text
sensorvärde → callback → villkor → styrkommando
```

**Kopplingsfråga:** Vilken del av koden måste ändras när turtlesims `Pose` ersätts av en fysisk avståndssensor, och vilken princip är oförändrad?

Svara med 1–2 meningar. Svaret ingår i inlämningen.

## Inlämning

Lämna in:

1. `lyssnare.py` från uppgift 1 och 2.
2. `stoppare.py`.
3. Tabellen från uppgift 3.
4. Dataflödet och svaren från uppgift 4.
5. Kort svar på kopplingsfrågan om robotmoppen.

## Vanliga problem

**Callbacken körs inte** — kontrollera att turtlesim körs och att workspacet är source:at.

**Sköldpaddan rycker** — stäng teleop när `stoppare.py` själv publicerar på `/turtle1/cmd_vel`.

**Fel message-typ** — `/turtle1/pose` använder `turtlesim/msg/Pose`, inte `geometry_msgs/msg/Pose`.
