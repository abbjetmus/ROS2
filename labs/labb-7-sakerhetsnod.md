# Labb 7: Robotmoppens säkerhetskedja i simulator

**Bygg samma ROS2-arkitektur som senare används på den fysiska robotmoppen**

**Förkunskaper:** Labb 1–6

## Syfte

Nu går vi från enskilda noder till ett litet robotsystem. Du ska bygga robotmoppens säkerhetskedja i turtlesim.

Du styr fortfarande roboten manuellt. Systemet ska bara kontrollera att ett framåtkommando inte skickas vidare när en sensor signalerar risk.

Det här är **inte autonom navigation**. Det är ett tydligt säkerhetslager som går att förstå, testa och senare återanvända på den fysiska roboten.

## Mål

Efter labben ska du kunna:

- förklara ett dataflöde mellan flera noder,
- skilja på ett rått och ett säkerhetskontrollerat körkommando,
- använda remapping för att koppla samma kod till olika robotar,
- använda en parameter,
- starta flera noder med en launch-fil,
- och beskriva vilka delar som byts när simulatorn ersätts av riktig hårdvara.

## Systemet vi bygger

```text
[turtle_teleop_key]
        │
        │ /robotmopp/cmd_vel_raw
        ▼
   [safety_node] ◄──── /robotmopp/safety_stop ──── [sim_safety_sensor]
        │                                                     ▲
        │ /robotmopp/cmd_vel                                  │
        ▼                                                     │ /turtle1/pose
   [turtlesim] ────────────────────────────────────────────────┘
```

### Nodernas ansvar

| Nod | Ansvar |
|---|---|
| `turtle_teleop_key` | Läser tangentbordet och skickar elevens önskade rörelse. |
| `sim_safety_sensor` | Använder turtlesims position som en simulerad säkerhetssensor. |
| `safety_node` | Stoppar framåtrörelse när `safety_stop` är `true`. |
| `turtlesim` | Spelar rollen som motorer och fysisk robot. |

### När robotmoppen byggs

| I labb 7 | På den riktiga robotmoppen |
|---|---|
| `turtle_teleop_key` | Tangentbord, joystick eller annan manuell styrning |
| `sim_safety_sensor` | Riktig avståndssensor och `sensor_node` |
| `safety_node` | **Samma säkerhetsnod återanvänds** |
| `turtlesim` | `motor_node`, motordrivare och motorer |

Det viktigaste gränssnittet är alltså topicsen. Om den riktiga sensorn publicerar samma typ av säkerhetssignal behöver resten av systemet inte skrivas om.

## Del 1: Hämta startkoden

I kursens repo finns:

```text
starter_code/labb7/
├── safety_node.py
├── sim_safety_sensor.py
└── labb7_robotmopp_sim.launch.py
```

Kopiera Python-filerna till:

```text
~/ros2_ws/src/min_turtle/min_turtle/
```

Kopiera launch-filen till:

```text
~/ros2_ws/src/min_turtle/launch/
```

Skapa launch-mappen om den saknas:

```bash
mkdir -p ~/ros2_ws/src/min_turtle/launch
```

> Börja med den fungerande startkoden. Målet är att läsa, köra, testa och göra små ändringar — inte att skriva hela systemet från tom fil.

## Del 2: Förstå `safety_node`

`SafetyNode` använder två inputs:

```text
/robotmopp/cmd_vel_raw    geometry_msgs/msg/Twist
/robotmopp/safety_stop    std_msgs/msg/Bool
```

och skickar ett output:

```text
/robotmopp/cmd_vel        geometry_msgs/msg/Twist
```

Kärnan är enkel:

```python
blockera_framat = self.safety_stop and msg.linear.x > 0.0

if blockera_framat:
    safe.linear.x = 0.0
```

Rotation och backning tillåts så att roboten kan ta sig bort från en risk.

Säkerhetsnoden startar i ett försiktigt läge och blockerar framåtkörning tills den har fått ett aktuellt sensormeddelande. Parametern `sensor_timeout_sec` gör också att framåt blockeras om sensorn slutar skicka data.

## Del 3: Förstå den simulerade sensorn

`sim_safety_sensor` lyssnar på:

```text
/turtle1/pose
```

Den kontrollerar om sköldpaddan är nära en kant och pekar mot kanten. Resultatet publiceras som:

```text
/robotmopp/safety_stop    true eller false
```

Parametern `margin` bestämmer hur stor säkerhetszonen är.

Du behöver inte själv ta fram geometrin i den här labben. Läs kommentarerna och förklara med egna ord vad sensorn försöker avgöra.

## Del 4: Registrera noderna

Kontrollera först att `package.xml` innehåller beroendet:

```xml
<depend>std_msgs</depend>
```

Om du skapade `min_turtle` med kommandot i den uppdaterade labb 4 finns det redan.

Öppna `setup.py` och lägg till under `console_scripts`:

```python
'safety_node = min_turtle.safety_node:main',
'sim_safety_sensor = min_turtle.sim_safety_sensor:main',
```

Se också till att launch-filer installeras. Högst upp i `setup.py` behövs:

```python
import os
from glob import glob
```

`data_files` ska innehålla:

```python
data_files=[
    ('share/ament_index/resource_index/packages',
        ['resource/' + package_name]),
    ('share/' + package_name, ['package.xml']),
    (os.path.join('share', package_name, 'launch'),
        glob('launch/*.launch.py')),
],
```

Bygg:

```bash
cd ~/ros2_ws
colcon build --packages-select min_turtle
source install/setup.bash
```

## Del 5: Starta systemet

Starta simulatorn, sensorn och säkerhetsnoden:

```bash
ros2 launch min_turtle labb7_robotmopp_sim.launch.py
```

Starta teleop i en separat terminal. Remapping gör att teleop skickar det **råa** kommandot till säkerhetsnoden i stället för direkt till roboten:

```bash
ros2 run turtlesim turtle_teleop_key --ros-args \
  --remap /turtle1/cmd_vel:=/robotmopp/cmd_vel_raw
```

Styr mot en kant. När den simulerade sensorn signalerar risk ska:

- framåt blockeras,
- rotation fortfarande fungera,
- bakåt fortfarande fungera.

## Del 6: Se dataflödet

Öppna en ny terminal:

```bash
ros2 node list
ros2 topic list
```

Studera noderna:

```bash
ros2 node info /safety_node
ros2 node info /sim_safety_sensor
```

Studera topicsen:

```bash
ros2 topic info /robotmopp/cmd_vel_raw
ros2 topic info /robotmopp/safety_stop
ros2 topic info /robotmopp/cmd_vel
```

Titta på signalerna:

```bash
ros2 topic echo /robotmopp/safety_stop
ros2 topic echo /robotmopp/cmd_vel
ros2 topic echo /robotmopp/status
```

Kör ett `echo` i taget och jämför vad som händer när du styr.

## Del 7: Ändra säkerhetszonen

Kontrollera parametern:

```bash
ros2 param get /sim_safety_sensor margin
```

Ändra den medan systemet kör:

```bash
ros2 param set /sim_safety_sensor margin 2.0
```

Testa igen. Robotens stoppzon ska nu börja längre från kanten.

## Del 8: Testa säkerhetsnoden utan simulator

Det här visar att säkerhetsnoden inte är beroende av turtlesim.

1. Stoppa `sim_safety_sensor` eller starta `safety_node` separat.
2. Publicera ett falskt sensormeddelande:

```bash
ros2 topic pub --rate 2 /robotmopp/safety_stop \
  std_msgs/msg/Bool "{data: false}"
```

3. Publicera ett framåtkommando i en annan terminal:

```bash
ros2 topic pub --rate 2 /robotmopp/cmd_vel_raw \
  geometry_msgs/msg/Twist \
  "{linear: {x: 1.0}, angular: {z: 0.0}}"
```

4. Läs output:

```bash
ros2 topic echo /robotmopp/cmd_vel
```

Byt sedan sensormeddelandet till:

```bash
ros2 topic pub --rate 2 /robotmopp/safety_stop \
  std_msgs/msg/Bool "{data: true}"
```

Framåtkommandot ska nu komma ut med `linear.x = 0.0`.

---

## GRUND — obligatoriskt

### Uppgift 1 — Rita systemet

Rita arkitekturen och markera:

- alla noder,
- alla topics,
- message-typen på varje topic,
- vilken nod som är publisher,
- och vilken nod som är subscriber.

### Uppgift 2 — Tre säkerhetstester

Genomför och dokumentera:

| Test | Sensor | Kommando | Förväntat resultat | Faktiskt resultat |
|---|---|---|---|---|
| 1 | `false` | framåt | framåt skickas vidare | ... |
| 2 | `true` | framåt | `linear.x` blir 0 | ... |
| 3 | `true` | rotation eller bakåt | kommandot tillåts | ... |

### Uppgift 3 — Testa parametern

Testa `margin` med exempelvis `0.6`, `1.2` och `2.0`. Skriv en mening om skillnaden för varje värde.

### Uppgift 4 — Förklara remapping

Förklara:

1. varför teleop inte längre får publicera direkt till roboten,
2. varför turtlesim i launch-filen lyssnar på `/robotmopp/cmd_vel`,
3. och hur detta gör det lättare att byta till en fysisk robot.

### Uppgift 5 — Kopplingen till robotmoppen

Svara kort:

- Vilken nod kan återanvändas utan att känna till sensorns fabrikat?
- Vilken nod måste anpassas till den fysiska sensorn?
- Vilken nod måste anpassas till motordrivaren?

## FÖRDJUPNING — frivilligt

Välj högst en:

### A — Nödstopp

Lägg till topicet `/robotmopp/emergency_stop`. När det är `true` ska all rörelse stoppas.

### B — Statusindikering

Ändra statusmeddelandet så att det tydligt visar `KLAR`, `STOPP` eller `VANTAR_PÅ_SENSOR`.

### C — Två säkerhetssensorer

Lägg till en andra bool-signal och blockera framåt om minst en sensor rapporterar risk.

## Inlämning

1. Arkitekturdiagrammet från uppgift 1.
2. Testtabellen från uppgift 2.
3. Resultat från parametertestet.
4. Svaren om remapping och robotmoppen.
5. En skärmdump där relevanta noder och topics syns.

## Vanliga problem

**Roboten rör sig inte alls** — kontrollera att `/robotmopp/safety_stop` faktiskt publiceras och att den har hunnit skicka minst ett meddelande.

**Teleop kringgår säkerheten** — teleop publicerar troligen fortfarande direkt till `/turtle1/cmd_vel`. Kontrollera remapping-kommandot.

**Turtlesim reagerar inte på `/robotmopp/cmd_vel`** — starta turtlesim via launch-filen så att dess `cmd_vel` blir remappat.

**Launch-filen hittas inte** — kontrollera `data_files`, bygg om och kör `source install/setup.bash`.

**Flera publishers krockar** — stäng gamla teleop-/topic-pub-processer med `Ctrl+C`.
