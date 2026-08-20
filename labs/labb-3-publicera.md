# Labb 3: Publicera egna kommandon

**Styr sköldpaddan direkt från terminalen med `ros2 topic pub` och `ros2 service call`**

**Förkunskaper:** Labb 1 och 2

## Syfte

Hittills har du använt färdiga noder för att styra sköldpaddan. Nu ska du skicka kommandon direkt från terminalen — utan teleop. Det är ett första steg mot att skriva egna noder i nästa labb.

## Mål

Efter labben ska du kunna:

- Publicera meddelanden manuellt med `ros2 topic pub`.
- Förstå skillnaden mellan **topic** och **service**.
- Anropa services med `ros2 service call`.
- Skapa nya sköldpaddor, ändra penna och teleportera med services.

## Förberedelse

Starta endast turtlesim (du behöver **inte** teleop i den här labben):

```bash
ros2 run turtlesim turtlesim_node
```

## Del 1: Publicera ett rörelsekommando

I en ny terminal:

```bash
ros2 topic pub --once /turtle1/cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 2.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
```

`--once` betyder "skicka en gång". Sköldpaddan ska röra sig kort framåt.

Skicka istället meddelandet kontinuerligt med 1 Hz:

```bash
ros2 topic pub --rate 1 /turtle1/cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 1.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 1.0}}"
```

Sköldpaddan kör i cirkel. Tryck `Ctrl+C` för att stoppa.

## Del 2: Services — fråga och svar

Topics används för **kontinuerlig data**. Services används för **engångsanrop** (gör X och ge mig svar).

Lista tillgängliga services:

```bash
ros2 service list
ros2 service list -t
```

Spawna en ny sköldpadda:

```bash
ros2 service call /spawn turtlesim/srv/Spawn \
  "{x: 2.0, y: 2.0, theta: 0.0, name: 'turtle2'}"
```

Teleportera den första sköldpaddan:

```bash
ros2 service call /turtle1/teleport_absolute turtlesim/srv/TeleportAbsolute \
  "{x: 5.5, y: 5.5, theta: 0.0}"
```

Ändra pennfärg och tjocklek:

```bash
ros2 service call /turtle1/set_pen turtlesim/srv/SetPen \
  "{r: 255, g: 0, b: 0, width: 5, off: 0}"
```

Rensa skärmen:

```bash
ros2 service call /clear std_srvs/srv/Empty "{}"
```

## Uppgifter

### Uppgift 1 — Snabb och långsam

Skicka `cmd_vel`-meddelanden med olika `linear.x`. Hitta:

- En hastighet där sköldpaddan rör sig nästan omärkligt långsamt.
- En hastighet där den åker tvärs över skärmen på under en sekund.
- Vad händer om du sätter `linear.x` till ett **negativt** värde?

### Uppgift 2 — Perfekt cirkel

Med rätt kombination av `linear.x` och `angular.z` kan du få sköldpaddan att rita en cirkel. Hitta värden så att den ritar:

- En liten cirkel (radie ≈ 1 enhet).
- En stor cirkel (radie ≈ 4 enheter).

> **Tips:** Radien blir `linear.x / angular.z`.

Skärmdump av båda cirklarna.

### Uppgift 3 — Regnbåge

Skriv en sekvens av kommandon (en lista i ditt dokument räcker) som:

1. Rensar skärmen.
2. Sätter pennan till röd.
3. Kör framåt en bit.
4. Sätter pennan till grön.
5. Kör framåt en bit till.
6. Sätter pennan till blå.
7. Kör framåt en bit till.

Kör sekvensen. Skärmdump.

### Uppgift 4 — Tre sköldpaddor

Spawna `turtle2` och `turtle3` på olika positioner. Kör `ros2 topic list`. Vilka nya topics finns? Försök publicera till `turtle2/cmd_vel` så att bara `turtle2` rör sig.

### Uppgift 5 — Skriv ditt initial automatiskt (utmaning)

Använd `teleport_absolute`, `set_pen` och `cmd_vel` för att rita första bokstaven i ditt namn **utan teleop**. Du får använda så många kommandon du vill. Spara sekvensen i en bash-fil och kör den med `bash mitt_namn.sh`.

Exempel på struktur:

```bash
#!/bin/bash
ros2 service call /turtle1/set_pen turtlesim/srv/SetPen "{off: 1}"
ros2 service call /turtle1/teleport_absolute turtlesim/srv/TeleportAbsolute "{x: 2.0, y: 8.0, theta: 0.0}"
ros2 service call /turtle1/set_pen turtlesim/srv/SetPen "{r: 0, g: 0, b: 255, width: 3, off: 0}"
ros2 topic pub --once /turtle1/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 5.0}}"
# osv...
```

## Inlämning

1. Hastigheter och observationer från uppgift 1.
2. Värden + skärmdumpar från uppgift 2.
3. Sekvensen och skärmdumpen från uppgift 3.
4. Svar från uppgift 4.
5. Bash-filen och skärmdump från uppgift 5.

## Användbara kommandon

```bash
ros2 topic pub --once <topic> <typ> "<yaml>"   # publicera en gång
ros2 topic pub --rate N <topic> <typ> "<yaml>" # publicera N ggr/sek
ros2 service list                              # alla services
ros2 service call <srv> <typ> "<yaml>"         # anropa service
ros2 interface show <typ>                      # struktur
```
