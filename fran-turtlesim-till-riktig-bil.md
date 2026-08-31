# Från turtlesim till riktig bil — hur ROS2 styr elevernas ESP8266-robotbil

Eleverna har byggt en robotbil med **ESP8266**, en TT-motor (PWM + riktningspinne), ett **180°-servo** för styrning och i flera fall en **HC-SR04** ultraljudssensor. Bilarna styrs idag via **MQTT** mot `mqtt-broker.cloud.mustini.com` (se t.ex. `240S/driverbot/`). Parallellt har de gått igenom labb 1–8 i ROS2 med turtlesim.

> **Labbarna finns nu:** upplägget nedan är genomfört som [labb 9 — Koppla ROS2 till din riktiga bil](labs/labb-9-koppla-bilen.md) och [labb 10 — Autonom bil](labs/labb-10-autonom-bil.md). Det här dokumentet är bakgrunden och motiveringen.

Det här dokumentet svarar på tre frågor:

1. Hur kan ett ROS2-program (t.ex. vägg-undvikaren från labb 7) styra den fysiska bilen och läsa dess sensorer?
2. Måste bilen byggas om med en Raspberry Pi?
3. Fungerar sensorer som är gjorda för Arduino även på Raspberry Pi?

---

## Kort svar

**Nej, de behöver inte byta till Raspberry Pi.** ROS2 kan inte köras *på* en ESP8266, men det behöver den inte heller. ROS2 körs på elevens dator (WSL2) precis som idag, och bilen blir en "drivrutin" som ROS2 pratar med över WiFi. Så är riktiga robotar oftast byggda: ROS på en dator, en mikrokontroller som sköter motorer och sensorer.

Eftersom bilarna **redan pratar MQTT** saknas bara en liten ROS2-nod som översätter mellan ROS2-topics och MQTT-topics. Då fungerar labb 7-koden nästan oförändrad mot den fysiska bilen.

---

## Alternativen

| | **A. ROS2 ↔ MQTT-brygga** (rekommenderas) | **B. micro-ROS på ESP32** | **C. Raspberry Pi på bilen** |
|---|---|---|---|
| Ny hårdvara | Ingen | ESP32 (~50–80 kr). ESP8266 stöds **inte** av micro-ROS | Pi 4/5 + strömförsörjning 5 V/3 A + ändå en motordrivare |
| Var kör ROS2 | På datorn (WSL2) | På datorn + "mikro-noder" på ESP32 | På Pi:n (Ubuntu 24.04) |
| Nätverk från WSL2 | Enkelt — bara utgående TCP till molnbrokern | micro-ROS-agent på datorn; ESP32 måste nå in i WSL2 → krångligt NAT | ROS2 DDS-discovery mellan WSL2 och Pi över LAN fungerar dåligt utan *mirrored networking* eller discovery server |
| Passar kursen | Ja — bygger direkt på labb 5–7 | Bra som fördjupning | Överkurs; mest tid går åt till Linux och nätverk |

Alternativ C är dessutom lite missvisande: även med en Raspberry Pi brukar man behålla en mikrokontroller för motorer och sensorer, eftersom Pi:n har svag PWM och saknar analoga ingångar (se sensoravsnittet längst ner).

---

## Så här fungerar alternativ A i praktiken

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

## Fungerar Arduino-sensorer på Raspberry Pi?

Delvis. Det avgörs av sensortyp och spänning, inte av märket "Arduino":

| Sensortyp | På Raspberry Pi | Kommentar |
|---|---|---|
| I2C / SPI (MPU6050, VL53L0X, BME280, OLED) | ✅ | Fungerar utmärkt, ofta bättre än på Arduino |
| Digitala (HC-SR04, knappar, DHT11, IR-hinderdetektor) | ⚠️ | Fungerar, men Pi:n tål bara 3,3 V på ingångarna — HC-SR04:s echo på 5 V kräver spänningsdelare. Tidsmätning i Python är också sämre än `pulseIn()` |
| Analoga (LDR, potentiometer, analog linjeföljare, MQ-gassensorer) | ❌ | Pi **saknar ADC** helt — kräver extern krets, t.ex. MCP3008 eller ADS1115 |
| Servo / PWM-motorer | ⚠️ | Bara 2 hårdvaru-PWM-kanaler; mjukvaru-PWM ryckar. Löses vanligen med ett PCA9685-kort |

ESP8266 är också 3,3 V, så sensorer som eleverna redan har fungerande på ESP:n är redan 3,3 V-kompatibla. Det som verkligen saknas på Pi:n är analoga ingångar och bra PWM — och det är precis därför man brukar behålla en mikrokontroller bredvid Pi:n även i "Raspberry Pi-robotar".

---

## Sammanfattning

- Behåll ESP8266-bilarna. ROS2 körs på datorn, bilen är en drivrutin.
- Lägg till en **ROS2 ↔ MQTT-brygga** (en Python-nod med `paho-mqtt`).
- Två små **firmware-ändringar**: ett kommando-topic med `hastighet,vinkel`, och avståndet publicerat som siffra. Plus dödmansgrepp.
- **Labb 9–10** är en naturlig fortsättning på labb 7: samma vägg-undvikare, fast på riktig hårdvara.
- **Skalan mellan världarna:** turtlesims 11 enheter ↔ arenans 2 m, dvs. ≈ 0,18 m/enhet. Längder räknas om rakt av; bilens fart och svängradie måste mätas — kalibreringsuppgifterna står i labb 10.
- Raspberry Pi tillför inget för den här kursen och skapar nya nätverks- och elektronikproblem. Vill ni senare ha "riktig" ROS2 på bilen är **ESP32 + micro-ROS** ett billigare och enklare steg än Pi.
