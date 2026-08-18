# Labb 2: Vad skickas mellan noderna?

**Topics och messages i ROS2**

**Förkunskaper:** Labb 1

## Syfte

I labb 1 såg du att två noder kommunicerade. Nu ska du undersöka **vad** som faktiskt skickas mellan dem. Du lär dig vad ett **topic** och ett **message** är, och hur du kan tjuvlyssna på datat med terminalverktyg.

## Mål

Efter labben ska du kunna:

- Lista och inspektera topics med `ros2 topic`.
- Förklara skillnaden mellan ett **topic** och ett **message**.
- Läsa innehållet i meddelanden i realtid med `ros2 topic echo`.
- Beskriva strukturen för `geometry_msgs/msg/Twist` och `turtlesim/msg/Pose`.

## Bakgrund

Ett **topic** är en namngiven kanal som noder kan skicka data på.
Ett **message** är datatypen som skickas på topicet.

I turtlesim finns bland annat:

| Topic | Message-typ | Beskrivning |
|---|---|---|
| `/turtle1/cmd_vel` | `geometry_msgs/msg/Twist` | Styrkommando (hastighet + rotation) |
| `/turtle1/pose` | `turtlesim/msg/Pose` | Sköldpaddans position och riktning |
| `/turtle1/color_sensor` | `turtlesim/msg/Color` | Färgen under sköldpaddan |

## Förberedelse

Starta turtlesim och teleop i två terminaler:

```bash
# Terminal 1
ros2 run turtlesim turtlesim_node

# Terminal 2
ros2 run turtlesim turtle_teleop_key
```

## Del 1: Lista topics

I en **tredje terminal**:

```bash
ros2 topic list
```

För att se vilken message-typ varje topic använder:

```bash
ros2 topic list -t
```

## Del 2: Tjuvlyssna med `echo`

```bash
ros2 topic echo /turtle1/pose
```

Texten i terminalen uppdateras hela tiden — det är sköldpaddans position som strömmar ut. Tryck `Ctrl+C` för att avsluta.

Lyssna även på styrkommandona:

```bash
ros2 topic echo /turtle1/cmd_vel
```

Tryck på piltangenter i teleop-terminalen och se hur meddelanden dyker upp.

## Del 3: Titta på message-strukturen

```bash
ros2 interface show geometry_msgs/msg/Twist
ros2 interface show turtlesim/msg/Pose
```

`Twist` består av två vektorer: `linear` (rörelse i x/y/z) och `angular` (rotation runt x/y/z). För en 2D-sköldpadda används bara `linear.x` (framåt) och `angular.z` (rotation).

## Del 4: Mät frekvens och bandbredd

```bash
ros2 topic hz /turtle1/pose
ros2 topic bw /turtle1/pose
```

Det första kommandot visar hur många meddelanden per sekund som skickas. Det andra visar bandbredden i bytes/sekund.

## Koppling till robotmoppen

Topics fungerar som robotmoppens interna signalkablar. Senare används bland annat:

```text
/robotmopp/cmd_vel_raw
/robotmopp/safety_stop
/robotmopp/cmd_vel
```

Varje topic har ett tydligt ansvar och en bestämd message-typ.

**Kopplingsfråga:** Vilket topic bör motorerna lyssna på — det råa kommandot eller det säkerhetskontrollerade kommandot? Förklara varför.

Svara med 1–2 meningar. Svaret ingår i inlämningen.

## Inlämning

1. Ifylld tabell från uppgift 1.
2. Svar och koordinater från uppgift 2.
3. Tabell över piltangenter och `cmd_vel`-värden från uppgift 3.
4. Svar på uppgift 4 och 5.
5. Kort svar på kopplingsfrågan om robotmoppen.

Uppgift 6 är frivillig fördjupning.

## Användbara kommandon

```bash
ros2 topic list             # alla topics
ros2 topic list -t          # alla topics + typer
ros2 topic echo <topic>     # läs ström
ros2 topic info <topic>     # publishers och subscribers
ros2 topic hz <topic>       # frekvens
ros2 topic bw <topic>       # bandbredd
ros2 interface show <typ>   # struktur för message
```
