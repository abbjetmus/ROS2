# Från turtlesim till riktig bil — hur ROS2 styr elevernas ESP8266-robotbil

Eleverna har byggt en robotbil med **ESP8266**, en TT-motor (PWM + riktningspinne), ett **180°-servo** för styrning och i flera fall en **HC-SR04** ultraljudssensor. Bilarna styrs idag via **MQTT** mot `mqtt-broker.cloud.mustini.com` (se t.ex. `240S/driverbot/`). Parallellt har de gått igenom labb 1–8 i ROS2 med turtlesim.

> **Labbarna finns nu:** upplägget nedan är genomfört som [labb 9 — Koppla ROS2 till din riktiga bil](labs/labb-9-koppla-bilen.md) och [labb 10 — Autonom bil](labs/labb-10-autonom-bil.md). Det här dokumentet är bakgrunden och motiveringen.

Det här dokumentet svarar på frågan: **hur kan ett ROS2-program (t.ex. vägg-undvikaren från labb 7) styra den fysiska bilen och läsa dess sensorer?**

---

## Kort svar

**Bilen behöver inte byggas om.** ROS2 kan inte köras *på* en ESP8266, men det behöver den inte heller. ROS2 körs på elevens dator (WSL2) precis som idag, och bilen blir en "drivrutin" som ROS2 pratar med över WiFi. Så är riktiga robotar oftast byggda: ROS på en dator, en mikrokontroller som sköter motorer och sensorer.

Eftersom bilarna **redan pratar MQTT** saknas bara en liten ROS2-nod som översätter mellan ROS2-topics och MQTT-topics. Då fungerar labb 7-koden nästan oförändrad mot den fysiska bilen.

---

## Lösningen: en brygga

All ROS2-kod körs på elevens dator i WSL2, precis som i labbarna. Det enda som tillkommer är en **bryggnod** som översätter mellan ROS2-topics och MQTT-topics — ingen ny hårdvara, och nätverket är enkelt eftersom både datorn och bilen bara gör utgående anslutningar till molnbrokern.

---

## Så här fungerar bryggan i praktiken

### Arkitektur

```
  Elevens dator (WSL2)                        Molnet                     Bilen
 ┌──────────────────────────────┐      ┌────────────────────┐      ┌─────────────────┐
 │ vagg_undvikare  ──/cmd_vel──►│      │                    │      │  ESP8266        │
 │      ▲                       │ MQTT │  mqtt-broker.      │ MQTT │  motor + servo  │
 │      │                bil_brygga◄──►│  cloud.mustini.com │◄────►│  HC-SR04        │
 │      └──/ultraljud───────────│      │                    │      │                 │
 └──────────────────────────────┘      └────────────────────┘      └─────────────────┘
```

- `bil_brygga` är en vanlig ROS2-nod (Python, rclpy) som också är en MQTT-klient.
- Elevens egen kod ser bara ROS2-topics: `/cmd_vel` (Twist) ut, `/ultraljud` (Range) in.

### 1. Firmware — liten ändring

Idag styrs bilen med separata topics (`<namn>/forward`, `<namn>/left` …) och sensorn skickar en sträng som `"Obstacle ahead!"`. För ROS2 är det smidigare att:

- lyssna på **ett** topic, t.ex. `<namn>/cmd`, med payload `"<hastighet>,<vinkel>"` (t.ex. `"600,-30"`):
  - hastighet −1023…1023 → `analogWrite(motorSpeed, abs(v))` + riktningspinne HIGH/LOW
  - vinkel −40…40 → `servo.write(90 + vinkel)`
- publicera avståndet som **siffra i cm** var 100 ms på `<namn>/distance`, istället för en text
- lägga in ett **dödmansgrepp**: stanna motorn om inget kommando kommit på 500 ms

Skiss på callback i firmware:

```cpp
void callback(char* topic, byte* payload, unsigned int length) {
  String msg;
  for (unsigned int i = 0; i < length; i++) msg += (char)payload[i];

  int komma = msg.indexOf(',');
  int hastighet = msg.substring(0, komma).toInt();   // -1023 .. 1023
  int vinkel    = msg.substring(komma + 1).toInt();  // -40 .. 40

  digitalWrite(motorDirection, hastighet >= 0 ? HIGH : LOW);
  analogWrite(motorSpeed, abs(hastighet));
  steeringServo.write(constrain(90 + vinkel, 50, 130));

  senasteKommando = millis();   // för dödmansgreppet i loop()
}

void loop() {
  client.loop();

  if (millis() - senasteKommando > 500) analogWrite(motorSpeed, 0);

  if (millis() - senasteMatning > 100) {
    senasteMatning = millis();
    float cm = getDistance();          // befintlig HC-SR04-funktion
    if (cm > 0) client.publish(distanceTopic, String(cm, 1).c_str());
  }
}
```

### 2. ROS2-bryggan

Installera MQTT-biblioteket i WSL (via `apt` — `pip` vägrar installera systemvitt på Ubuntu 24.04):

```bash
sudo apt install python3-paho-mqtt
```

Skapa `src/min_turtle/min_turtle/bil_brygga.py`:

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from sensor_msgs.msg import Range
import paho.mqtt.client as mqtt

NAMN = '24lucsil'   # elevens eget prefix, samma som i firmware


class BilBrygga(Node):
    def __init__(self):
        super().__init__('bil_brygga')

        # MQTT-sidan
        self.mqtt = mqtt.Client()
        self.mqtt.on_message = self.fran_bilen
        self.mqtt.connect('mqtt-broker.cloud.mustini.com', 1883)
        self.mqtt.subscribe(f'{NAMN}/distance')
        self.mqtt.loop_start()

        # ROS2-sidan
        self.create_subscription(Twist, '/cmd_vel', self.till_bilen, 10)
        self.avstand_pub = self.create_publisher(Range, '/ultraljud', 10)
        self.get_logger().info(f'Brygga igång för {NAMN}')

    def till_bilen(self, msg: Twist):
        # linear.x  -1..1  ->  PWM -1023..1023
        # angular.z -1..1  ->  servovinkel -40..40 grader
        hastighet = int(max(-1.0, min(1.0, msg.linear.x)) * 1023)
        vinkel = int(max(-1.0, min(1.0, msg.angular.z)) * 40)
        self.mqtt.publish(f'{NAMN}/cmd', f'{hastighet},{vinkel}')

    def fran_bilen(self, client, userdata, m):
        r = Range()
        r.header.stamp = self.get_clock().now().to_msg()
        r.header.frame_id = 'ultraljud'
        r.radiation_type = Range.ULTRASOUND
        r.min_range = 0.02
        r.max_range = 4.0
        r.range = float(m.payload) / 100.0   # cm -> meter
        self.avstand_pub.publish(r)


def main():
    rclpy.init()
    rclpy.spin(BilBrygga())
    rclpy.shutdown()
```

Lägg till i `setup.py` under `entry_points` precis som i labb 5:

```python
'bil_brygga = min_turtle.bil_brygga:main',
```

Bygg och starta:

```bash
cd ~/ros2_ws && colcon build && source install/setup.bash
ros2 run min_turtle bil_brygga
```

Testa utan egen kod först — styr bilen med tangentbordet:

```bash
sudo apt install ros-$ROS_DISTRO-teleop-twist-keyboard
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -p repeat_rate:=10.0
```

(`repeat_rate` gör att senaste kommandot skickas om 10 ggr/s, annars stoppar dödmansgreppet bilen efter 500 ms.)

Och titta på sensorn:

```bash
ros2 topic echo /ultraljud
```

### 3. Elevens kod från labb 7

Vägg-undvikaren ändras på tre ställen:

| Turtlesim (labb 7) | Riktig bil |
|---|---|
| `self.create_publisher(Twist, '/turtle1/cmd_vel', 10)` | `self.create_publisher(Twist, '/cmd_vel', 10)` |
| `self.create_subscription(Pose, '/turtle1/pose', self.cb, 10)` | `self.create_subscription(Range, '/ultraljud', self.cb, 10)` |
| `nara_vagg()` jämför x/y mot kanterna | `return msg.range < 0.3` |

Dessutom bör hastigheterna skalas ner (`linear.x = 0.5` istället för `2.0`, `angular.z = 1.0` blir fullt servoutslag). Allt annat — noder, topics, tillståndsmaskinen, `ros2 topic echo`, `ros2 node list` — gäller rakt av.

### Praktiska noteringar

- **Latens:** via molnbrokern ~50–200 ms. Helt okej för en långsam bil, men därför är dödmansgreppet i firmware viktigt.
- **Lokal broker?** Man kan köra `mosquitto` på datorn, men då måste ESP:n nå in i WSL2, vilket kräver *mirrored networking* i Windows 11. Enklast är att behålla molnbrokern.
- **Färdiga paket** finns (`ros-jazzy-mqtt-client`), men en egen brygga på ~40 rader är mer pedagogisk och enklare att felsöka.
- **Flera bilar** samtidigt fungerar utan krock så länge varje elev använder sitt eget MQTT-prefix och kör sin egen brygga.

---

## Sammanfattning

- Behåll ESP8266-bilarna. ROS2 körs på datorn, bilen är en drivrutin.
- Lägg till en **ROS2 ↔ MQTT-brygga** (en Python-nod med `paho-mqtt`).
- Två små **firmware-ändringar**: ett kommando-topic med `hastighet,vinkel`, och avståndet publicerat som siffra. Plus dödmansgrepp.
- **Labb 9–10** är en naturlig fortsättning på labb 7: samma vägg-undvikare, fast på riktig hårdvara.
- **Skalan mellan världarna:** turtlesims 11 enheter ↔ arenans 2 m, dvs. ≈ 0,18 m/enhet. Längder räknas om rakt av; bilens fart och svängradie måste mätas — kalibreringsuppgifterna står i labb 10.
