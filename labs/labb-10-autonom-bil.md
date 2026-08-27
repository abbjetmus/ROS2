# Labb 10: Autonom bil

**Vägg-undvikaren från labb 7 — på riktigt**

**Förkunskaper:** Labb 1–9

## Syfte

I labb 7 byggde du en tillståndsmaskin som fick sköldpaddan att köra runt utan att fastna. Nu ska samma idé få din riktiga bil att köra runt utan att krocka. Koden blir förvånansvärt lik — men verkligheten är inte turtlesim, och de skillnaderna är den egentliga lärdomen i den här labben.

Målet är konkret: **bilen ska på egen hand täcka en yta på 2 × 2 m.** Med en whiteboardpenna tejpad under bilen ska det finnas pennspår överallt när den är klar — precis som sköldpaddans pennspår på skärmen, fast på golvet.

## Mål

Efter labben ska du kunna:

- Flytta en nod från simulering till riktig hårdvara och förklara vad som ändrades.
- Hantera **brusig sensordata** (filtrering).
- Designa en tillståndsmaskin för en robot som **inte kan svänga på stället**.
- Göra beteendet inställbart med **parametrar** och starta hela systemet med en **launch-fil**.
- Mäta hur stor del av en yta roboten täckt och använda måttet för att jämföra inställningar.

## Bakgrund: vad är annorlunda mot turtlesim?

| | Turtlesim (labb 7) | Riktig bil |
|---|---|---|
| Vad noden vet | Exakt position `x, y, theta` | Bara avståndet rakt fram (`/ultraljud`) |
| Sensorn | Perfekt, 60 Hz | HC-SR04: 10 Hz, spikar, missar mjuka/sneda ytor |
| Svänga | På stället (`angular.z` utan `linear.x`) | Bara medan bilen rullar — servot vrider hjulen, motorn måste gå |
| Tid | Ett kommando verkar direkt | 50–200 ms latens, motorn tar tid att stanna |
| Fel | Sköldpaddan glider längs väggen | Bilen krockar |

Konsekvensen för tillståndsmaskinen: `VANDA`-tillståndet från labb 7 fungerar inte. En bil som står stilla med hjulen vridna svänger inte. Istället:

```
   ┌──────────────┐   avstånd < STOPP    ┌──────────────┐
   │  KOR_FRAMAT  │ ───────────────────► │  BACKA_SVANG │
   │  rakt fram   │                      │  backa med   │
   │              │ ◄─────────────────── │  fullt utslag│
   └──────────────┘   BACKA_TID sekunder └──────────────┘
```

## Del 1: Säkerhet först — hinderstopp

Innan bilen får köra själv ska den kunna **stanna** själv. Skapa `src/min_turtle/min_turtle/hinder_stopp.py` (jämför `stoppare.py` i labb 6):

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from sensor_msgs.msg import Range

STOPP_AVSTAND = 0.30   # meter
FART = 0.4


class HinderStopp(Node):
    def __init__(self):
        super().__init__('hinder_stopp')
        self.publisher = self.create_publisher(Twist, '/cmd_vel', 10)
        self.create_subscription(Range, '/ultraljud', self.cb, 10)
        self.senaste_avstand = None
        self.timer = self.create_timer(0.1, self.publicera)

    def cb(self, msg: Range):
        self.senaste_avstand = msg.range

    def publicera(self):
        cmd = Twist()
        if self.senaste_avstand is None:
            pass                              # ingen sensordata än -> stå still
        elif self.senaste_avstand < STOPP_AVSTAND:
            self.get_logger().info(f'Stop! Hinder på {self.senaste_avstand:.2f} m')
        else:
            cmd.linear.x = FART
        self.publisher.publish(cmd)


def main():
    rclpy.init()
    rclpy.spin(HinderStopp())
    rclpy.shutdown()
```

Lägg märke till två saker som skiljer sig från labb 6:

- Noden publicerar från en **timer** (10 Hz), inte från sensor-callbacken. Då fortsätter kommandon att skickas även om sensorn tystnar — och dödmansgreppet i firmware slår till om noden tystnar. Två lager säkerhet.
- Innan första sensorvärdet kommit **står bilen still**. En robot som inte vet något ska inte röra sig.

Registrera i `setup.py` (`'hinder_stopp = min_turtle.hinder_stopp:main'`), bygg, och kör med bryggan igång i en annan terminal:

```bash
ros2 run min_turtle bil_brygga --ros-args -p namn:=<prefix>
ros2 run min_turtle hinder_stopp
```

Ställ bilen på golvet riktad mot en vägg. Den ska köra fram och stanna ungefär 30 cm före. Fungerar det inte — kör **inte** vidare till del 2.

## Del 2: Hinder-undvikare

Skapa `src/min_turtle/min_turtle/hinder_undvikare.py`:

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from sensor_msgs.msg import Range

KOR_FRAMAT = 'kor_framat'
BACKA_SVANG = 'backa_svang'

STOPP_AVSTAND = 0.30   # meter
BACKA_TID = 1.5        # sekunder
FART = 0.4


class HinderUndvikare(Node):
    def __init__(self):
        super().__init__('hinder_undvikare')
        self.publisher = self.create_publisher(Twist, '/cmd_vel', 10)
        self.create_subscription(Range, '/ultraljud', self.cb, 10)
        self.tillstand = KOR_FRAMAT
        self.backa_start = None
        self.senaste_avstand = None
        self.timer = self.create_timer(0.1, self.publicera)

    def cb(self, msg: Range):
        self.senaste_avstand = msg.range
        if self.tillstand == KOR_FRAMAT and msg.range < STOPP_AVSTAND:
            self.tillstand = BACKA_SVANG
            self.backa_start = self.get_clock().now()
            self.get_logger().info(f'Hinder på {msg.range:.2f} m — backar och svänger')

    def sekunder_sedan(self, tidpunkt) -> float:
        return (self.get_clock().now() - tidpunkt).nanoseconds / 1e9

    def publicera(self):
        cmd = Twist()
        if self.senaste_avstand is None:
            self.publisher.publish(cmd)       # stå still tills sensorn lever
            return

        if self.tillstand == KOR_FRAMAT:
            cmd.linear.x = FART
        elif self.sekunder_sedan(self.backa_start) < BACKA_TID:
            cmd.linear.x = -FART
            cmd.angular.z = 1.0               # fullt utslag åt vänster
        else:
            self.tillstand = KOR_FRAMAT
            self.get_logger().info('Fri väg — kör framåt')

        self.publisher.publish(cmd)


def main():
    rclpy.init()
    rclpy.spin(HinderUndvikare())
    rclpy.shutdown()
```

Jämför med `vagg_undvikare.py` från labb 7: samma struktur (callback byter tillstånd, timern publicerar), men sensorvillkoret är ett enda tal och vändningen är tidsstyrd eftersom bilen inte har någon `theta` att titta på.

Registrera, bygg, kör. Bilen ska köra runt i rummet, backa och svänga vid hinder.

## Del 3: Arena, penna och täckningsmått

Det som i turtlesim är pennspåret och arenan bygger du nu på golvet.

| Del | Så gör du |
|---|---|
| **Arena** | 2 × 2 m papper på golvet (byggpapp på rulle eller hopptejpade ark). Kartongväggar runt om, minst 15 cm höga så att ultraljudet ser dem. |
| **Rutnät** | Rita ett rutnät med 25 cm rutor = 8 × 8 = **64 rutor**. |
| **Penna** | Whiteboard- eller tuschpenna tejpad under bilens mitt, spetsen mot golvet med lite fjädring (en bit skumgummi räcker). |
| **Start** | Bilen står i ett hörn, riktad in mot arenan. |

**Mätprotokoll** (samma varje gång, annars går resultaten inte att jämföra):

1. Starta bryggan och noden. Ta tid från det att bilen börjar röra sig.
2. Kör tills **alla rutor har pennspår** eller tills **3 minuter** gått — det som kommer först.
3. Räkna rutor med spår. Täckning = rutor med spår / 64.
4. Anteckna även antal krockar (bilen nuddar väggen) och antal gånger den stått still mer än 5 sekunder.

| Körning | Inställningar | Tid | Täckning | Krockar | Fast |
|---|---|---|---|---|---|
| 1 | STOPP 0.30, BACKA 1.5, FART 0.4 | | | | |
| 2 | | | | | |

Byt papper (eller rita på baksidan) mellan körningarna. Fotografera pennspåret efter varje körning — det är din motsvarighet till skärmdumpen av turtlesim.

## Uppgifter

### Uppgift 1 — Trimma

Testa `STOPP_AVSTAND` = 0.15, 0.30 och 0.60 samt `BACKA_TID` = 0.5, 1.5 och 3.0 enligt mätprotokollet i del 3 — **ändra en sak i taget**. Fyll i tabellen: tid, täckning, krockar, fast. Vilka värden ger högst täckning på kortast tid hos dig, och hur beror de på bilens fart?

### Uppgift 2 — Filtrera sensorn

HC-SR04 ger ibland enstaka felaktiga värden (t.ex. 300 cm när det egentligen är 20, eller tvärtom). Ett enda spikvärde ska inte få bilen att backa. Ändra `cb()` så att noden sparar de **tre senaste** avstånden och använder **medianen** när den fattar beslut.

```python
from collections import deque
import statistics

self.historik = deque(maxlen=3)
# i cb:
self.historik.append(msg.range)
avstand = statistics.median(self.historik)
```

Logga både råvärdet och medianen under en körning. Skärmdump där du ser ett spikvärde som medianen tar bort.

### Uppgift 3 — Parametrar

Gör `STOPP_AVSTAND`, `BACKA_TID` och `FART` till ROS2-parametrar (labb 4, uppgift 4) så att du kan trimma utan att bygga om:

```bash
ros2 run min_turtle hinder_undvikare --ros-args -p stopp_avstand:=0.4 -p fart:=0.3
```

### Uppgift 4 — Launch-fil

Skriv `launch/bil.launch.py` som startar bryggan (med ditt prefix som argument) och hinder-undvikaren med ett kommando. Se labb 8 för hur launch-filer registreras i `setup.py`. Parametrar skickas så här:

```python
Node(
    package='min_turtle',
    executable='bil_brygga',
    parameters=[{'namn': '24lucsil'}],
),
```

```bash
ros2 launch min_turtle bil.launch.py
```

### Uppgift 5 — Välj en utmaning

**a) Växla svängriktning.** Bilen svänger alltid åt vänster när den backar. Låt den växla håll varannan gång, eller slumpa. Fastnar den mer eller mindre sällan i hörn?

**b) Mjuk inbromsning.** Istället för full fart fram till `STOPP_AVSTAND` — låt farten vara proportionell mot avståndet (jämför `jaga_mal.py` i labb 6): `fart = min(FART, K * (avstand - STOPP_AVSTAND))`. Bilen ska sakta in mjukt och stanna precis framför hindret.

**c) Fastnat-detektor.** Om avståndet inte ändrats på 3 sekunder trots att bilen "kör framåt" — bilen sitter fast (t.ex. mot något sensorn inte ser). Backa längre än vanligt. Jämför labb 7, uppgift 5.

**d) Mönster istället för slump.** Byt ut `KOR_FRAMAT` mot en **spiral** (`angular.z` som långsamt minskar med tiden, jämför labb 5 uppgift 2) eller, om du har ett gyro (MPU6050), **banor** med riktiga 90°-svängar. Mät täckningen efter 2 minuter med samma protokoll och jämför med slumpvarianten. Vilken täcker mest per minut?

**e) Fjärrstyrning + autonomi.** Skriv en `vaktmastare`-nod som lyssnar på **både** `/cmd_vel_teleop` (från teleop, byt topic med `--ros-args -r cmd_vel:=cmd_vel_teleop`) och `/ultraljud`, och som släpper igenom teleop-kommandon till `/cmd_vel` **utom** framåt-kommandon när det är hinder inom `STOPP_AVSTAND`. En förare som inte kan köra in i väggen.

## Inlämning

1. Tabellen från uppgift 1 + foto på pennspåret från din bästa körning, bredvid en skärmdump av turtlesim-spåret från labb 7 eller 8.
2. `hinder_undvikare.py` med medianfilter och parametrar (uppgift 2–3) + skärmdumpen från uppgift 2.
3. `bil.launch.py`.
4. En av utmaningarna i uppgift 5.
5. Visa bilen täcka arenan autonomt för någon av lärarna.
6. **Reflektion** (10–15 meningar), som ska svara på:
   - Vad var skillnaden mellan att skriva noden för turtlesim och för bilen? Vilka delar av labb 7-koden kunde du behålla rakt av?
   - Hur lång tid tog det att täcka 2 × 2 m? Hur lång tid skulle ett klassrum ta med samma robot — och är det rimligt?
   - Vilket problem i verkligheten hade du inte förutsett?
   - Vilken **en** ytterligare sensor skulle förbättra bilen mest, och hur skulle du koppla in den i ROS2 (vilket topic, vilken meddelandetyp)?

## Vanliga problem

**Bilen står helt still** — får noden sensordata? `ros2 topic hz /ultraljud`. Om inget kommer: kolla bryggan och firmware enligt labb 9.

**Bilen backar hela tiden** — sensorn ser något nära (golvet, en kabel, bilens egen front?). Logga `msg.range` och kolla vad sensorn faktiskt ser. Vinkla sensorn en aning uppåt.

**Bilen krockar innan den hinner stanna** — höj `STOPP_AVSTAND` eller sänk `FART`. Räkna: vid 0.5 m/s och 200 ms latens har bilen hunnit 10 cm innan kommandot ens kommer fram.

**Bilen svänger inte när den backar** — några motordrivare klarar inte riktningsbyte utan en kort paus. Lägg in ett kort `STANNA`-tillstånd (0.3 s med `linear.x = 0`) mellan `KOR_FRAMAT` och `BACKA_SVANG`.

**Två noder publicerar på `/cmd_vel`** — precis som i labb 6: stäng teleop när den autonoma noden kör, annars får bilen blandade kommandon.
