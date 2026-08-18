# Labb 5: Publisher-nod i Python

**Skriv en egen ROS2-nod som skickar styrkommandon**

**Förkunskaper:** Labb 1–4

## Syfte

I labb 3 skickade du `cmd_vel`-meddelanden manuellt från terminalen. Nu skriver du en Python-nod som gör samma sak automatiskt — en **publisher**.

Fokus i den här labben är inte avancerad robotlogik. Fokus är att förstå hur en egen nod skapar och publicerar ROS2-meddelanden.

## Mål

Efter labben ska du kunna:

- Skapa en publisher med `create_publisher`.
- Använda `create_timer` för att köra en funktion regelbundet.
- Skapa ett `Twist`-meddelande i Python.
- Publicera meddelandet på `/turtle1/cmd_vel`.
- Ändra ett värde och förutse hur robotens rörelse påverkas.

## Del 1: Aktivera workspacet

```bash
cd ~/ros2_ws
source install/setup.bash
```

Starta turtlesim i en separat terminal:

```bash
ros2 run turtlesim turtlesim_node
```

## Del 2: Skapa publisher-noden

Skapa filen:

```text
src/min_turtle/min_turtle/cirkel.py
```

Lägg in:

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

### Vad är viktigast i koden?

| Kod | Vad den gör |
|---|---|
| `create_publisher(...)` | Skapar en publisher på ett topic. |
| `create_timer(0.1, ...)` | Kör funktionen var 0,1 sekund. |
| `msg = Twist()` | Skapar ett nytt rörelsemeddelande. |
| `msg.linear.x` | Hastighet framåt/bakåt. |
| `msg.angular.z` | Rotation vänster/höger. |
| `publish(msg)` | Skickar meddelandet. |
| `rclpy.spin(node)` | Håller noden igång. |

## Del 3: Registrera noden

Öppna `src/min_turtle/setup.py` och lägg till:

```python
'cirkel = min_turtle.cirkel:main',
```

under `console_scripts`.

Bygg och kör:

```bash
cd ~/ros2_ws
colcon build --packages-select min_turtle
source install/setup.bash
ros2 run min_turtle cirkel
```

Sköldpaddan ska börja köra i en cirkel.

## Del 4: Kontrollera att ROS2 ser din nod

Med cirkel-noden igång, öppna en annan terminal:

```bash
ros2 node list
ros2 node info /cirkel_nod
ros2 topic echo /turtle1/cmd_vel
```

Försök identifiera:

- nodens namn,
- vilket topic den publicerar på,
- vilken message-typ som används.

---

## Koppling till robotmoppen

Publisher-noden i den här labben motsvarar senare en styrnod som skickar önskad rörelse på:

```text
/robotmopp/cmd_vel_raw
```

Kommandot går först till säkerhetsnoden och därefter till motorerna. Styrningen ska alltså inte köra motorerna direkt.

**Kopplingsfråga:** Vad kan hända om en controller publicerar direkt till motorn och kringgår säkerhetsnoden?

Svara med 1–2 meningar. Svaret ingår i inlämningen.

## Inlämning

Lämna in:

1. `cirkel.py`.
2. Kort svar från uppgift 1.
3. Värden + skärmdumpar från uppgift 2.
4. Skärmdump och förklaring från uppgift 3.
5. Dataflödet från uppgift 4.
6. Kort svar på kopplingsfrågan om robotmoppen.

## Vanliga problem

**Sköldpaddan rör sig inte** — kontrollera att turtlesim körs och att topicet heter `/turtle1/cmd_vel`.

**Ändringar syns inte** — bygg om och source:a igen:

```bash
colcon build --packages-select min_turtle
source install/setup.bash
```

**Två noder styr samtidigt** — stäng `turtle_teleop_key` när du testar din egen publisher.
