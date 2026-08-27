# Labb 8: Slutprojekt — autonom sköldpadda

**Eget projekt — bygg din egen autonoma robot**

**Förkunskaper:** Labb 1–7

## Syfte

I slutprojektet sätter du ihop allt du lärt dig och bygger en **egen** autonom sköldpadda. Du väljer själv vilket beteende din robot ska ha, så länge det uppfyller projektets minimikrav.

## Lärandemål

Eleven ska visa att hen kan:

- Designa ett robotsystem med flera noder.
- Implementera minst en publisher och minst en subscriber i eget Python-package.
- Använda **services** och/eller **parametrar**.
- Reflektera över robotens beteende och dess kopplingar till verkliga robotar.

## Minimikrav

Ditt projekt **måste** innehålla:

1. **Eget package** byggt med `colcon` i `~/ros2_ws`.
2. **Minst två noder** som kommunicerar via ett topic.
3. **Minst en subscriber** som påverkar robotens beteende (sensorvärden ⇒ beslut).
4. **Minst en service-anrop** (t.ex. `set_pen`, `teleport_absolute`, `spawn`).
5. **En launch-fil** som startar hela systemet med ett kommando.
6. **README.md** i ditt package som beskriver projektet (se mallen längst ned).

## Projektförslag

Välj ett av förslagen eller kom på något eget (godkänns av läraren).

### Förslag A — Förföljaren

Två sköldpaddor: `turtle1` rör sig längs en förutbestämd bana (t.ex. en cirkel). `turtle2` försöker hela tiden köra mot `turtle1`. Logga avståndet mellan dem var sekund.

### Förslag B — Konstnären

En nod som ritar en figur du själv designat — t.ex. ett mönster av cirklar, en stjärnhimmel, eller ditt namn skrivet med olika färger. Använd `set_pen` och `teleport_absolute` för att variera färg och hoppa mellan delar.

### Förslag C — Smart städare

Sköldpaddan ska "städa" arenan genom att besöka så stor area som möjligt på 60 sekunder. Använd vägg-undvikaren från labb 7 men förbättra den, t.ex. med slumpmässig vändning eller minne av besökta områden.

### Förslag D — Hinderbana

Spawna 3–5 sköldpaddor som står stilla och fungerar som hinder. En sjätte sköldpadda ska navigera från startpunkten till en målpunkt utan att komma närmare än 1.5 enheter från något hinder.

### Förslag E — Trafikljus

Två sköldpaddor som korsar varandras vägar. En `trafikljus`-nod publicerar `röd`/`grön` på ett eget topic. Sköldpaddorna stannar vid en linje när ljuset är rött.

## Steg-för-steg

### 1. Planering

- Välj projektidé.
- Beskriv den kortfattat i din repository på Github.

### 2. Implementation

- Skapa eller återanvänd ditt package från labb 4.
- Implementera noderna en i taget och testa dem var för sig.
- Använd `git` för att hålla ordning på din kod (rekommenderas).

### 3. Launch-fil

Skapa filen `src/min_turtle/launch/projekt.launch.py`:

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(package='turtlesim', executable='turtlesim_node'),
        Node(package='min_turtle', executable='min_nod'),
        # ... fler noder
    ])
```

Lägg till i `setup.py`:

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
ros2 launch min_turtle projekt.launch.py
```

### 4. Inlämning
Slutprojektet redovisas genom en inlämning av kod och visat resultatet för någon av lärarna. 

## README-mall

Skriv en `README.md` i ditt package med följande korta sektioner:

### 1. Projektbeskrivning

Vad gör din robot? Vilken idé valde du och varför?

### 2. Systemarkitektur

Rita ett diagram över dina noder och topics. Exempel:

```
/teleop_egen ──/turtle1/cmd_vel──► /turtlesim
                                       │
                                       ▼
                              /turtle1/pose
                                       │
                                       ▼
                              /min_styrare
```

### 3. Så kör man projektet

Vilka kommandon behövs för att bygga och starta systemet (t.ex. `colcon build` och `ros2 launch ...`).

### 4. Reflektion

- Vad var svårast?
- Vad skulle behövas för att din kod skulle fungera på en riktig robot (t.ex. en städrobot)?
- Vilka begränsningar har turtlesim jämfört med en verklig miljö?

## Lycka till!

Du har redan byggt: en publisher, en subscriber, en tillståndsmaskin, och du har använt services och parametrar. Slutprojektet handlar om att sätta ihop bitarna till något du själv har designat. Var ambitiös — men välj något du hinner bli klar med.

## Nästa steg

Slutprojektet avslutar simuleringsdelen. I [labb 9](labb-9-koppla-bilen.md) kopplar du ROS2 till din riktiga robotbil från Driverbot-projektet, och i [labb 10](labb-10-autonom-bil.md) flyttar du vägg-undvikaren från labb 7 över till den. Den kod du skrivit här fungerar med små ändringar även där.
