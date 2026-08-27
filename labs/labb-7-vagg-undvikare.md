# Labb 7: Autonom vägg-undvikare

**Bygg en sköldpadda som rör sig själv utan att krocka i kanterna**

**Förkunskaper:** Labb 1–6

## Syfte

Du ska kombinera allt du lärt dig och bygga ett **autonomt beteende**. Sköldpaddan ska köra runt på egen hand utan att fastna mot väggarna. Det är en miniatyrversion av hur en riktig städrobot eller självkörande robot agerar.

## Mål

Efter labben ska du kunna:

- Strukturera en nod som ett **enkelt tillståndsmaskin** (states).
- Skilja mellan **sensorlogik** (vad ser jag?) och **kontrolllogik** (vad gör jag?).
- Hantera kantfall som "fastnat i hörn" eller "kan inte vända fritt".

## Bakgrund: tillstånd

En typisk vägg-undvikare har minst två tillstånd:

| Tillstånd | Vad noden gör |
|---|---|
| `KOR_FRAMAT` | Publicera `linear.x = 2.0`. Byta till `VANDA` när man närmar sig en vägg. |
| `VANDA` | Publicera `angular.z = 1.5`. Byta tillbaka till `KOR_FRAMAT` när man pekar bort från väggen. |

```
   ┌──────────────┐    nära vägg     ┌────────┐
   │ KOR_FRAMAT   │ ───────────────► │ VANDA  │
   │              │                  │        │
   │              │ ◄─────────────── │        │
   └──────────────┘  klar med vändning└────────┘
```

## Del 1: Skelett

Skapa `src/min_turtle/min_turtle/vagg_undvikare.py`:

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from turtlesim.msg import Pose


KOR_FRAMAT = 'kor_framat'
VANDA = 'vanda'

MARGINAL = 1.5   # hur nära kanten innan vi vänder
MIN_X, MAX_X = 0.0, 11.0
MIN_Y, MAX_Y = 0.0, 11.0


class VaggUndvikare(Node):
    def __init__(self):
        super().__init__('vagg_undvikare')
        self.publisher = self.create_publisher(Twist, '/turtle1/cmd_vel', 10)
        self.create_subscription(Pose, '/turtle1/pose', self.cb, 10)
        self.tillstand = KOR_FRAMAT
        self.timer = self.create_timer(0.1, self.publicera)
        self.senaste_pose: Pose | None = None

    def cb(self, msg: Pose):
        self.senaste_pose = msg
        if self.tillstand == KOR_FRAMAT and self.nara_vagg(msg):
            self.tillstand = VANDA
            self.get_logger().info('Vägg upptäckt — vänder')
        elif self.tillstand == VANDA and not self.nara_vagg(msg):
            self.tillstand = KOR_FRAMAT
            self.get_logger().info('Fri väg — kör framåt')

    def nara_vagg(self, p: Pose) -> bool:
        return (
            p.x < MIN_X + MARGINAL or p.x > MAX_X - MARGINAL
            or p.y < MIN_Y + MARGINAL or p.y > MAX_Y - MARGINAL
        )

    def publicera(self):
        cmd = Twist()
        if self.tillstand == KOR_FRAMAT:
            cmd.linear.x = 2.0
        else:
            cmd.angular.z = 1.5
        self.publisher.publish(cmd)


def main():
    rclpy.init()
    rclpy.spin(VaggUndvikare())
    rclpy.shutdown()
```

Registrera, bygg och kör. Sköldpaddan ska bukta omkring inne i arenan utan att fastna.

## Uppgifter

### Uppgift 1 — Hitta en bra marginal

Testa `MARGINAL` = 0.5, 1.5 och 3.0. Beskriv vad som händer i varje fall. Vilken marginal tycker du fungerar bäst, och varför?

### Uppgift 2 — Smart vändning

I `vanda`-läget vänder noden alltid åt samma håll. Det betyder att sköldpaddan ibland vänder mot väggen. Förbättra logiken så att den **vänder bort från väggen**:

- Använd `msg.theta` (sköldpaddans riktning) och var den befinner sig.
- Om sköldpaddan är nära **höger** vägg (`x > MAX_X - MARGINAL`), välj rotationsriktning så att den vänder mot vänster.
- Generalisera för alla fyra väggar.

Tips: räkna ut målriktningen `mal_theta` (till mitten av arenan, t.ex. `atan2(5.5 - msg.y, 5.5 - msg.x)`) och vänd åt det håll som minskar vinkelfelet snabbast.

### Uppgift 3 — Pennspår

Lägg till en initiering som sätter pennfärgen till något du gillar:

```python
from turtlesim.srv import SetPen
self.pen_client = self.create_client(SetPen, '/turtle1/set_pen')
self.pen_client.wait_for_service()
req = SetPen.Request()
req.r, req.g, req.b, req.width, req.off = 0, 200, 255, 3, 0
self.pen_client.call_async(req)
```

Kör i 30 sekunder. Skärmdump av spåret.

### Uppgift 4 — Flera sköldpaddor (utmaning)

Spawna `turtle2`. Skriv din nod så att den tar emot turtlens namn som **parameter**:

```python
self.declare_parameter('turtle', 'turtle1')
turtle = self.get_parameter('turtle').value
self.publisher = self.create_publisher(Twist, f'/{turtle}/cmd_vel', 10)
self.create_subscription(Pose, f'/{turtle}/pose', self.cb, 10)
```

Starta två instanser av samma nod i två terminaler — en med `turtle:=turtle1`, en med `turtle:=turtle2`. Skärmdump där båda kör autonomt samtidigt.

### Uppgift 5 — Krockdetektor (utmaning)

Trots vägg-undvikaren kan sköldpaddan ibland fastna i ett hörn. Lägg till logik som upptäcker det och tvingar en längre vändning:

- Spara sköldpaddans position varje sekund.
- Om den inte rört sig mer än 0.5 enheter på 3 sekunder — räkna det som "fastnat".
- I så fall, vänd i 4 sekunder.

## Inlämning

1. Bordet med tester från uppgift 1.
2. `vagg_undvikare.py` med smart vändning (uppgift 2).
3. Skärmdump av pennspåret (uppgift 3).
4. Antingen uppgift 4 eller 5 + skärmdump.
5. En kort reflektion (5–10 meningar): vad var svårast? Var det något du fick lösa på ett annat sätt än du först tänkt? Vad skulle behövas för att samma nod skulle fungera på en riktig robot? (Spara svaret — i labb 9–10 gör vi precis det med din Driverbot-bil.)

## Tips för felsökning

**Sköldpaddan svänger för evigt** — ditt villkor för att gå tillbaka till `KOR_FRAMAT` triggas aldrig. Logga `theta` och `nara_vagg` i callbacken för att förstå varför.

**Sköldpaddan stannar mitt i arenan** — du publicerar troligen ett tomt `Twist` när tillståndet inte matchar. Kolla att du täcker alla fall.

**Två kommandokällor krockar** — om du har teleop igång samtidigt får sköldpaddan motstridiga kommandon. Stäng teleop.
