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

## Koppling till robotmoppen

`geometry_msgs/msg/Twist` används inte bara i turtlesim. Samma typ av meddelande kan beskriva önskad rörelse för robotmoppen:

- `linear.x` — framåt eller bakåt,
- `angular.z` — vänster eller höger.

Den fysiska `motor_node` översätter sedan dessa två värden till signaler för vänster och höger motor.

**Kopplingsfråga:** Varför är det praktiskt att styrningen skickar ett allmänt `Twist`-meddelande i stället för att känna till exakt motormodell?

Svara med 1–2 meningar. Svaret ingår i inlämningen.

## Inlämning

1. Hastigheter och observationer från uppgift 1.
2. Värden + skärmdumpar från uppgift 2.
3. Sekvensen och skärmdumpen från uppgift 3.
4. Svar från uppgift 4.
5. Kort svar på kopplingsfrågan om robotmoppen.

Uppgift 5 är frivillig fördjupning.

## Användbara kommandon

```bash
ros2 topic pub --once <topic> <typ> "<yaml>"   # publicera en gång
ros2 topic pub --rate N <topic> <typ> "<yaml>" # publicera N ggr/sek
ros2 service list                              # alla services
ros2 service call <srv> <typ> "<yaml>"         # anropa service
ros2 interface show <typ>                      # struktur
```
