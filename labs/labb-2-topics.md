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

## Uppgifter

### Uppgift 1 — Topic-tabell

Fyll i tabellen för **alla** topics du hittade med `ros2 topic list -t`:

| Topic | Message-typ | Vad tror du den används till? |
|---|---|---|
| ... | ... | ... |

### Uppgift 2 — Avläs position

Kör `ros2 topic echo /turtle1/pose` och styr sköldpaddan med teleop. Skriv ner sköldpaddans `x` och `y` när den är:

- I startposition.
- Längst upp till höger på skärmen.
- Längst ner till vänster på skärmen.

Vilket koordinatsystem använder turtlesim? Var ligger origo (`0,0`)?

### Uppgift 3 — Tolka cmd_vel

Lyssna på `/turtle1/cmd_vel` och tryck på varje piltangent i teleop ett kort ögonblick. Skriv ner vilka värden som ändras i meddelandet för:

- Pil upp
- Pil ner
- Pil vänster
- Pil höger

### Uppgift 4 — Frekvensanalys

Kör `ros2 topic hz /turtle1/pose`. Skriv ner frekvensen. Kör sedan samma kommando för `/turtle1/cmd_vel` **medan du håller en piltangent nedtryckt**. Vilken skillnad ser du, och varför?

### Uppgift 5 — Identifiera kopplingen

Använd `ros2 topic info /turtle1/cmd_vel`. Vilken nod är publisher? Vilken är subscriber? Rita en pil-skiss av dataflödet.

### Uppgift 6 — Krockdetektor (utmaning)

Använd `ros2 topic echo /turtle1/pose` och styr sköldpaddan rakt in i en vägg. Vad händer med `x`, `y` och `theta` när sköldpaddan krockar? Skriv en hypotes om hur turtlesim hanterar väggar.

## Inlämning

1. Ifylld tabell från uppgift 1.
2. Svar och koordinater från uppgift 2.
3. Tabell över piltangenter och cmd_vel-värden från uppgift 3.
4. Svar på uppgift 4, 5 och 6.

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
