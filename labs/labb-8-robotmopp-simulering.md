# Labb 8: Robotmoppen i simulering

**Bygg hela städprogrammet i turtlesim — innan det flyttas till din riktiga bil**

**Förkunskaper:** Labb 1–7

## Syfte

Kursens projekt är en **robotmopp**: din Driverbot-bil ska med en whiteboardpenna under sig täcka en arena på 2 × 2 m med pennspår. I den här labben bygger du hela städprogrammet färdigt i turtlesim först. Det är billig felsökning — en bugg i simuleringen kostar en omstart, en bugg på golvet kostar en krockad bil.

Det du bygger här är inte en övning som slängs bort. I labb 9 kopplar du ROS2 till bilen, och i labb 10 kör **samma program** på golvet. Sköldpaddans arena är din arenas simulering:

| I turtlesim (denna labb) | På golvet (labb 9–10) |
|---|---|
| Arenan 11 × 11 enheter | Arenan 2 × 2 m (1 enhet ≈ 0,18 m) |
| Rutnät 8 × 8 = 64 rutor (1,375 enheter per ruta) | Rutnät 8 × 8 = 64 rutor (25 cm per ruta) |
| Sköldpaddans pennspår | Whiteboardpennan under bilen |
| Täckningsmätar-nod räknar besökta rutor | Du räknar rutor med pennspår för hand |
| Teleop styr sköldpaddan | Teleop styr bilen (via MQTT-bryggan) |

**Nivåerna i projektet** (gäller den riktiga bilen i labb 10, men repeteras här i simulering):

- **Nivå 1:** du styr bilen **manuellt** från ROS2 (teleop) och täcker **alla rutor** med pennan.
- **Nivå 2:** bilen täcker ytan **autonomt** (slumpstädning med hinder-undvikning).
- **Nivå 3:** en **smartare strategi** (spiral eller styr-mot-oritat) som mätbart slår slumpen.

## Mål

Efter labben ska du kunna:

- Mäta **täckning** med en egen subscriber-nod (rutnät över arenan).
- Städa arenan manuellt med teleop och jämföra med ett autonomt beteende.
- Bygga en städ-nod med tillståndsmaskinen `KOR_FRAMAT ⇄ BACKA_SVANG` — samma som bilen använder.
- Starta hela systemet med en **launch-fil**.

## Del 1: Täckningsmätaren

I turtlesim finns inget papper att räkna rutor på — men noden vet exakt var sköldpaddan är. Skriv en nod som delar arenan i 8 × 8 rutor och håller reda på vilka som besökts.

Skapa `src/min_turtle/min_turtle/tackning.py`:

```python
import rclpy
from rclpy.node import Node
from turtlesim.msg import Pose

RUTOR = 8        # 8 x 8 = 64 rutor, precis som på den riktiga arenan
ARENA = 11.0     # turtlesims arena är ca 11 x 11 enheter


class Tackning(Node):
    def __init__(self):
        super().__init__('tackning')
        self.create_subscription(Pose, '/turtle1/pose', self.cb, 10)
        self.besokta = set()
        self.start = self.get_clock().now()
        self.klar = False
        self.create_timer(2.0, self.rapportera)

    def cb(self, msg: Pose):
        kolumn = min(RUTOR - 1, max(0, int(msg.x / ARENA * RUTOR)))
        rad = min(RUTOR - 1, max(0, int(msg.y / ARENA * RUTOR)))
        self.besokta.add((rad, kolumn))

    def sekunder(self) -> float:
        return (self.get_clock().now() - self.start).nanoseconds / 1e9

    def rapportera(self):
        antal = len(self.besokta)
        self.get_logger().info(
            f'Täckning: {antal}/{RUTOR * RUTOR} rutor ({antal / (RUTOR * RUTOR):.0%}) '
            f'efter {self.sekunder():.0f} s')
        if antal == RUTOR * RUTOR and not self.klar:
            self.klar = True
            self.get_logger().info(f'*** ALLA rutor täckta på {self.sekunder():.0f} sekunder! ***')


def main():
    rclpy.init()
    rclpy.spin(Tackning())
    rclpy.shutdown()
```

Registrera i `setup.py` (`'tackning = min_turtle.tackning:main'`) och bygg med `colcon build` som vanligt (labb 4).

> **Obs:** mätaren räknar en ruta som besökt när sköldpaddans *mittpunkt* varit där. På golvet räknas en ruta när den har pennspår — samma idé, mätt på olika sätt.

## Del 2: Städa manuellt — nivå 1 på skärmen

Nu gör du i simulering exakt det som nivå 1 kräver på golvet: styr manuellt och berör **alla** rutor.

Tre terminaler:

```bash
ros2 run turtlesim turtlesim_node
ros2 run min_turtle tackning
ros2 run turtlesim turtle_teleop_key
```

Kör runt med piltangenterna tills mätaren rapporterar 64/64. Ta tiden (mätaren loggar den åt dig) och ta en skärmdump av pennspåret.

Fundera medan du kör: vilken **strategi** använder du själv? Rader fram och tillbaka som en gräsklippare? Spiral utifrån och in? Det du gör med fingrarna nu är det som ska bli kod i del 3 — och samma sak gör du med den riktiga bilen i labb 10.

## Del 3: Städa autonomt — med bilens tillståndsmaskin

Vägg-undvikaren från labb 7 svänger på stället (`VANDA`). **Det kan inte den riktiga bilen** — den är servostyrd och svänger bara medan den rullar. För att programmet ska gå att flytta till bilen härmar vi bilens beteende redan i simuleringen:

```
   ┌──────────────┐    nära vägg         ┌──────────────┐
   │  KOR_FRAMAT  │ ───────────────────► │  BACKA_SVANG │
   │  rakt fram   │                      │  backa med   │
   │              │ ◄─────────────────── │  fullt utslag│
   └──────────────┘   BACKA_TID sekunder └──────────────┘
```

Skapa `src/min_turtle/min_turtle/stadare.py`:

```python
import random

import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from turtlesim.msg import Pose

KOR_FRAMAT = 'kor_framat'
BACKA_SVANG = 'backa_svang'

MARGINAL = 1.5     # enheter — jämför STOPP_AVSTAND 0.30 m i labb 10 (1.5 x 0.18 ≈ 0.27 m)
BACKA_TID = 1.5    # sekunder
FART = 2.0         # enheter/s ≈ 0.36 m/s på golvet
MIN_X, MAX_X = 0.0, 11.0
MIN_Y, MAX_Y = 0.0, 11.0


class Stadare(Node):
    def __init__(self):
        super().__init__('stadare')
        self.publisher = self.create_publisher(Twist, '/turtle1/cmd_vel', 10)
        self.create_subscription(Pose, '/turtle1/pose', self.cb, 10)
        self.tillstand = KOR_FRAMAT
        self.backa_start = None
        self.svang_riktning = 1.0
        self.timer = self.create_timer(0.1, self.publicera)

    def cb(self, msg: Pose):
        if self.tillstand == KOR_FRAMAT and self.nara_vagg(msg):
            self.tillstand = BACKA_SVANG
            self.backa_start = self.get_clock().now()
            self.svang_riktning = random.choice([-1.0, 1.0])
            self.get_logger().info('Vägg — backar och svänger')

    def nara_vagg(self, p: Pose) -> bool:
        return (
            p.x < MIN_X + MARGINAL or p.x > MAX_X - MARGINAL
            or p.y < MIN_Y + MARGINAL or p.y > MAX_Y - MARGINAL
        )

    def sekunder_sedan(self, tidpunkt) -> float:
        return (self.get_clock().now() - tidpunkt).nanoseconds / 1e9

    def publicera(self):
        cmd = Twist()
        if self.tillstand == KOR_FRAMAT:
            cmd.linear.x = FART
        elif self.sekunder_sedan(self.backa_start) < BACKA_TID:
            cmd.linear.x = -FART / 2
            cmd.angular.z = 1.5 * self.svang_riktning
        else:
            self.tillstand = KOR_FRAMAT
        self.publisher.publish(cmd)


def main():
    rclpy.init()
    rclpy.spin(Stadare())
    rclpy.shutdown()
```

Jämför med `vagg_undvikare.py` från labb 7 — strukturen är identisk (callbacken byter tillstånd, timern publicerar), men vändningen är **tidsstyrd baklängeskörning med sväng**, precis som `hinder_undvikare.py` som du skriver för bilen i labb 10. Slumpen i svängriktningen gör att den inte fastnar i samma mönster.

Registrera, bygg, och kör med täckningsmätaren igång. Hur lång tid tar slumpstädaren jämfört med dig?

## Del 4: Launch-fil — allt med ett kommando

Tre terminaler för att starta ett system är opraktiskt. Skapa `src/min_turtle/launch/mopp.launch.py`:

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(package='turtlesim', executable='turtlesim_node'),
        Node(package='min_turtle', executable='stadare'),
        Node(package='min_turtle', executable='tackning'),
    ])
```

Lägg till i `setup.py` så att launch-filen följer med bygget:

```python
import os
from glob import glob
# ...
data_files=[
    ('share/ament_index/resource_index/packages', ['resource/min_turtle']),
    ('share/' + package_name, ['package.xml']),
    (os.path.join('share', package_name, 'launch'), glob('launch/*.launch.py')),
],
```

Kör med:

```bash
colcon build --packages-select min_turtle
source install/setup.bash
ros2 launch min_turtle mopp.launch.py
```

I labb 10 skriver du en likadan launch-fil som startar bryggan och städlogiken för den riktiga bilen.

## Uppgifter

### Uppgift 1 — Manuellt mot autonomt

Fyll i tabellen. Kör varje variant minst två gånger (starta om turtlesim mellan körningarna så pennspåret nollställs):

| Körning | Strategi | Tid till 64/64 (eller täckning efter 3 min) |
|---|---|---|
| 1 | Manuell (teleop) | |
| 2 | Manuell, annan strategi | |
| 3 | Autonom slumpstädare | |
| 4 | Autonom slumpstädare | |

Varför skiljer sig slumpstädarens tider mellan körningarna, när din manuella tid är ungefär densamma?

### Uppgift 2 — Parametrar

Gör `FART`, `BACKA_TID` och `MARGINAL` till ROS2-parametrar (labb 4, uppgift 4) så att du kan trimma utan att bygga om:

```bash
ros2 run min_turtle stadare --ros-args -p fart:=3.0 -p backa_tid:=1.0
```

Testa minst tre kombinationer och anteckna täckningstiden. Vilken är bäst? I labb 10 gör du om exakt samma trimning med bilen — då tar varje körning flera minuter, så det lönar sig att ha en känsla för parametrarna redan nu.

### Uppgift 3 — Spiral

En robotmopp som kör i **spiral** täcker systematiskt istället för slumpmässigt. Lägg till ett tillstånd `SPIRAL` som startar med hög `angular.z` som långsamt minskar (jämför labb 5, uppgift 2), och gå över till slumpstädning när spiralen når en vägg. Mät med täckningsmätaren: täcker spiralen mer än slumpen per minut?

### Uppgift 4 — Styr mot det oritade (utmaning)

Slumpstädarens svaghet är att den gärna åker där den redan varit. Din täckningsmätare vet ju vilka rutor som saknas! Bygg ihop logiken (i en nod, eller via ett eget topic): när städaren ska välja svängriktning, välj den riktning som pekar mot närmaste **obesökta** ruta.

Detta är simuleringens motsvarighet till nivå 3-beteendet på bilen — där gör ljussensorn samma jobb genom att se om golvet redan har pennspår.

### Uppgift 5 — Realistisk fart (utmaning)

`FART = 2.0` enheter/s motsvarar ca 0,36 m/s på golvet (skalfaktorn 0,18 m/enhet). När du i labb 10 mätt din bils verkliga fart: kör om simuleringen med `fart := din fart / 0.18` och jämför pennspåren. Då ser du i förväg hur din bil kommer att bete sig.

## Inlämning

1. Skärmdump av det **manuella** pennspåret (64/64) + din tid.
2. Skärmdump av det **autonoma** pennspåret + tabellen från uppgift 1.
3. `tackning.py` och `stadare.py` med parametrar (uppgift 2).
4. `mopp.launch.py`.
5. Uppgift 3 (spiral) med mätresultat, **eller** uppgift 4 (styr mot oritat).
6. Reflektion (5–10 meningar): vilken strategi vann? Vad tror du blir annorlunda när samma program ska köra en riktig bil på golvet? (Spara svaret — du jämför med facit i labb 10.)

## Vanliga problem

**Mätaren visar 64/64 direkt** — du startade den efter att du redan kört runt ett tag. Starta om turtlesim och mätaren tillsammans (eller använd launch-filen).

**Städaren fastnar i ett hörn** — höj `BACKA_TID` eller `MARGINAL`. Jämför labb 7, uppgift 5 (krockdetektorn) — samma lösning fungerar här.

**Launch-filen hittas inte** — glömde du `data_files`-raden i `setup.py`, eller att köra `colcon build` + `source install/setup.bash` efteråt?

**Sköldpaddan rycker fram och tillbaka** — både teleop och städaren publicerar på `/turtle1/cmd_vel`. Kör aldrig båda samtidigt.

## Nästa steg

Städprogrammet är klart och testat. I [labb 9](labb-9-koppla-bilen.md) kopplar du ROS2 till din riktiga bil via MQTT-bryggan, och i [labb 10](labb-10-autonom-bil.md) flyttar du städlogiken till golvet: nivå 1 är att göra del 2 med riktiga bilen (teleop, alla rutor), högre nivåer är att låta bilen göra del 3–4 själv.
