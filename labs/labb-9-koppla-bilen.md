# Labb 9: Koppla ROS2 till din riktiga bil

**Från turtlesim till ESP8266 — samma `cmd_vel`, riktig motor**

**Förkunskaper:** Labb 1–7 samt en fungerande robotbil (ESP8266, motor, servo, HC-SR04) från Driverbot-projektet

## Syfte

Hittills har dina noder styrt en sköldpadda på skärmen. Nu ska **exakt samma sorts noder** styra din riktiga bil. Det gör du inte genom att köra ROS2 på bilen (det går inte på en ESP8266) utan genom att bygga en **brygga**: en ROS2-nod på din dator som översätter `Twist`-meddelanden till MQTT-kommandon som bilen redan förstår, och som tar emot bilens sensordata och publicerar dem som ROS2-topics.

Det är så riktiga robotar oftast är byggda: ROS på en dator, en mikrokontroller som sköter motorer och sensorer.

## Mål

Efter labben ska du kunna:

- Förklara varför ROS2 körs på datorn och inte på mikrokontrollern.
- Anpassa bilens firmware till ett enkelt, generellt **kommandoprotokoll** över MQTT.
- Skriva en ROS2-nod som samtidigt är en MQTT-klient (bryggan).
- Konvertera mellan ROS2-meddelanden (`Twist`, `Range`) och råa värden (PWM, servovinkel, cm).
- Styra bilen med tangentbordet via ROS2 och läsa dess ultraljudssensor med `ros2 topic echo`.

## Bakgrund: arkitekturen

```
  Din dator (WSL2)                              Molnet                      Bilen
 ┌────────────────────────────────┐      ┌────────────────────┐      ┌─────────────────┐
 │ teleop / din nod ──/cmd_vel──► │      │                    │      │  ESP8266        │
 │                                │ MQTT │  mqtt-broker.      │ MQTT │  motor + servo  │
 │                       bil_brygga ◄───►│  cloud.mustini.com │◄────►│  HC-SR04        │
 │                                │      │                    │      │                 │
 │ ros2 topic echo ◄──/ultraljud──│      └────────────────────┘      └─────────────────┘
 └────────────────────────────────┘
```

| Del | Vad den gör |
|---|---|
| **Firmware** (ESP8266) | Lyssnar på `<namn>/cmd`, styr motor och servo. Skickar avstånd på `<namn>/distance` 10 ggr/s. |
| **`bil_brygga`** (ROS2-nod) | Prenumererar på `/cmd_vel`, publicerar MQTT. Prenumererar på MQTT, publicerar `/ultraljud`. |
| **Dina noder** | Ser bara ROS2-topics. Vet inte att det finns MQTT över huvud taget. |

### Protokollet mellan brygga och bil

| MQTT-topic | Riktning | Payload | Exempel |
|---|---|---|---|
| `<namn>/cmd` | dator → bil | `"<hastighet>,<vinkel>"` där hastighet är −1023…1023 (PWM, negativt = bakåt) och vinkel är −40…40 grader | `"600,-30"` |
| `<namn>/distance` | bil → dator | avstånd i cm som decimaltal | `"37.2"` |

`<namn>` är ditt eget prefix, samma som du använde i Driverbot-projektet. Alla i klassen delar samma broker, så prefixet är det som skiljer din bil från de andras.

> **Vilken broker?** Labben är skriven för skolans broker `mqtt-broker.cloud.mustini.com` (ingen inloggning). Använder din klass **Maqiatto** (`maqiatto.com`) gäller: användarnamn = din e-post, och `<namn>` **måste** vara din e-post (t.ex. `fornamn.efternamn@hitachigymnasiet.se`), eftersom Maqiatto bara tillåter topics som börjar med den. Ange inloggning i firmware (`MQTT_USER`/`MQTT_PASS`) och i bryggan (`-p anvandare:=... -p losenord:=...`).

### Två saker som inte fanns i turtlesim

- **Dödmansgrepp.** Om WiFi eller bryggan dör mitt i en körning får bilen inte fortsätta framåt. Därför stannar firmware motorn om inget kommando kommit på 500 ms.
- **Latens.** Ett kommando går via molnet och tar 50–200 ms. Det märks inte vid låg fart, men det är en av anledningarna till att dödmansgreppet ligger i firmware och inte i ROS2.

## Del 1: Ny firmware

Du ska ersätta styr-delen av din Driverbot-firmware med mallen nedan. Behåll dina pinnar och din motorfunktion.

> Mallen använder biblioteket **EspMQTTClient** (Arduino IDE → Library Manager → "EspMQTTClient" av Patrick Lapointe). Använder din gamla kod `PubSubClient` direkt? Se rutan efter koden.

Skapa en ny sketch `ros_bil.ino`:

```cpp
#include "EspMQTTClient.h"
#include <Servo.h>

// ==================== ANPASSA DESSA ====================
#define WIFI_SSID     "Hitachigymnasiet_2.4"
#define WIFI_PASSWORD "..."
#define MQTT_NAMN     "24lucsil"   // ditt prefix — samma i ROS2-bryggan
#define MQTT_BROKER   "mqtt-broker.cloud.mustini.com"   // eller "maqiatto.com"
#define MQTT_USER     ""           // samma som i din gamla firmware ("" om inget)
#define MQTT_PASS     ""

#define MOTOR_PWM  D1   // hastighet (PWM)
#define MOTOR_DIR  D3   // riktning (HIGH = framåt)
#define SERVO_PIN  D4
#define TRIG_PIN   D5
#define ECHO_PIN   D6

const int SERVO_CENTER = 90;          // vinkel där hjulen pekar rakt fram
const int SERVO_MAX_UTSLAG = 40;      // grader åt varje håll
// =======================================================

const unsigned long DODMANS_MS = 500;   // stanna om inget kommando på så här länge
const unsigned long MATNING_MS = 100;   // hur ofta avståndet skickas

EspMQTTClient client(
  WIFI_SSID, WIFI_PASSWORD,
  MQTT_BROKER,
  MQTT_USER, MQTT_PASS,
  MQTT_NAMN "-bil",                     // klientnamn, måste vara unikt på brokern
  1883
);

Servo styrServo;
const String cmdTopic  = String(MQTT_NAMN) + "/cmd";
const String distTopic = String(MQTT_NAMN) + "/distance";

unsigned long senasteKommando = 0;
unsigned long senasteMatning = 0;

// ---------- Motor och servo ----------
// Det här är den ENDA funktionen du behöver ändra om din motordrivare
// styrs på ett annat sätt (t.ex. L298N med två IN-pinnar).
void korMotor(int hastighet) {          // -1023 .. 1023
  hastighet = constrain(hastighet, -1023, 1023);
  digitalWrite(MOTOR_DIR, hastighet >= 0 ? HIGH : LOW);
  analogWrite(MOTOR_PWM, abs(hastighet));
}

void styr(int vinkel) {                 // -40 .. 40 grader, positivt = vänster
  vinkel = constrain(vinkel, -SERVO_MAX_UTSLAG, SERVO_MAX_UTSLAG);
  styrServo.write(SERVO_CENTER + vinkel);
}

// ---------- Ultraljud ----------
float matAvstand() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long tid = pulseIn(ECHO_PIN, HIGH, 30000);   // max 30 ms ≈ 5 m
  if (tid == 0) return -1;                     // inget eko
  return tid * 0.034 / 2.0;                    // cm
}

// ---------- MQTT ----------
void taEmotKommando(const String& payload) {   // "600,-30"
  int komma = payload.indexOf(',');
  if (komma < 0) return;
  int hastighet = payload.substring(0, komma).toInt();
  int vinkel    = payload.substring(komma + 1).toInt();

  korMotor(hastighet);
  styr(vinkel);
  senasteKommando = millis();
}

void onConnectionEstablished() {               // anropas av EspMQTTClient
  client.subscribe(cmdTopic, taEmotKommando);
  Serial.println("MQTT ansluten. Lyssnar på " + cmdTopic);
}

// ---------- Setup och loop ----------
void setup() {
  Serial.begin(115200);

  pinMode(MOTOR_PWM, OUTPUT);
  pinMode(MOTOR_DIR, OUTPUT);
  analogWriteRange(1023);                      // ESP8266 core 3.x har annars 0-255
  korMotor(0);

  styrServo.attach(SERVO_PIN);
  styr(0);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  client.enableDebuggingMessages();            // WiFi/MQTT-status i Serial Monitor
}

void loop() {
  client.loop();
  unsigned long nu = millis();

  // Dödmansgrepp
  if (nu - senasteKommando > DODMANS_MS) {
    korMotor(0);
  }

  // Skicka avstånd 10 ggr/s
  if (nu - senasteMatning > MATNING_MS) {
    senasteMatning = nu;
    float cm = matAvstand();
    if (cm > 0 && client.isConnected()) {
      client.publish(distTopic, String(cm, 1));
    }
  }
}
```

> **Använder du `PubSubClient`?** Behåll din `setup_wifi()` och `reconnect()`. Byt ut din `callback()` mot logiken i `taEmotKommando()` (payload kommer som `byte*` — bygg en `String` av den först), prenumerera bara på `<namn>/cmd` i `reconnect()`, och lägg in dödmansgreppet och avståndspubliceringen i `loop()`.

Ladda upp, öppna Serial Monitor (115200 baud) och vänta tills det står `MQTT ansluten`.

## Del 2: Testa firmware utan ROS2

Precis som i labb 3 (där du skickade `Twist` från terminalen innan du skrev en nod) ska du först testa bilen "för hand". Installera MQTT-verktyg i WSL:

```bash
sudo apt install -y mosquitto-clients
```

Lyssna på sensorn i en terminal (byt `24lucsil` mot ditt prefix; lägg till `-u <användare> -P <lösenord>` om din firmware använder inloggning):

```bash
mosquitto_sub -h mqtt-broker.cloud.mustini.com -t 24lucsil/distance
```

(Maqiatto: `mosquitto_sub -h maqiatto.com -u <epost> -P <lösenord> -t "<epost>/distance"`.)

Du ska se avstånd rulla in ~10 ggr/s. Håll handen framför sensorn och se värdet ändras.

Skicka ett kommando i en annan terminal — **ställ bilen på en låda så att hjulen snurrar fritt**:

```bash
mosquitto_pub -h mqtt-broker.cloud.mustini.com -t 24lucsil/cmd -m "500,0"
```

Hjulen ska snurra i cirka en halv sekund och sedan stanna — det är dödmansgreppet. Prova `"500,30"`, `"-500,0"` och `"0,-40"`. Gå inte vidare förrän alla fyra gör det du förväntar dig.

## Del 3: Bryggnoden

### Beroenden

Bryggan behöver ett MQTT-bibliotek för Python och ROS2-meddelandetypen `Range`:

```bash
sudo apt install -y python3-paho-mqtt
```

Öppna `src/min_turtle/package.xml` och lägg till bredvid de andra `<depend>`-raderna:

```xml
<depend>sensor_msgs</depend>
```

### Koden

Skapa `src/min_turtle/min_turtle/bil_brygga.py`:

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from sensor_msgs.msg import Range
import paho.mqtt.client as mqtt

MAX_PWM = 1023      # linear.x = 1.0  ->  full fart
MAX_VINKEL = 40     # angular.z = 1.0 ->  fullt servoutslag


class BilBrygga(Node):
    def __init__(self):
        super().__init__('bil_brygga')

        # Parametrar (jämför labb 4, uppgift 4)
        self.declare_parameter('namn', 'andra_mig')
        self.declare_parameter('broker', 'mqtt-broker.cloud.mustini.com')
        self.declare_parameter('anvandare', '')
        self.declare_parameter('losenord', '')
        namn = self.get_parameter('namn').value
        broker = self.get_parameter('broker').value
        self.cmd_topic = f'{namn}/cmd'
        self.dist_topic = f'{namn}/distance'

        # ROS2-sidan
        self.create_subscription(Twist, '/cmd_vel', self.till_bilen, 10)
        self.avstand_pub = self.create_publisher(Range, '/ultraljud', 10)

        # MQTT-sidan
        self.mqtt = mqtt.Client()
        anvandare = self.get_parameter('anvandare').value
        if anvandare:
            self.mqtt.username_pw_set(anvandare, self.get_parameter('losenord').value)
        self.mqtt.on_connect = self.mqtt_ansluten
        self.mqtt.on_message = self.fran_bilen
        self.mqtt.connect(broker, 1883)
        self.mqtt.loop_start()      # MQTT kör i egen tråd, ROS2 spinner i huvudtråden

        self.get_logger().info(
            f'Brygga: /cmd_vel -> {self.cmd_topic},  {self.dist_topic} -> /ultraljud'
        )

    # ---- MQTT-callbacks ----
    def mqtt_ansluten(self, client, userdata, flags, rc):
        client.subscribe(self.dist_topic)
        self.get_logger().info('Ansluten till MQTT-brokern')

    def fran_bilen(self, client, userdata, m):
        try:
            cm = float(m.payload)
        except ValueError:
            return
        r = Range()
        r.header.stamp = self.get_clock().now().to_msg()
        r.header.frame_id = 'ultraljud'
        r.radiation_type = Range.ULTRASOUND
        r.field_of_view = 0.26      # ca 15 grader
        r.min_range = 0.02
        r.max_range = 4.0
        r.range = cm / 100.0        # cm -> meter (ROS2 använder alltid SI-enheter)
        self.avstand_pub.publish(r)

    # ---- ROS2-callback ----
    def till_bilen(self, msg: Twist):
        fart = max(-1.0, min(1.0, msg.linear.x))
        svang = max(-1.0, min(1.0, msg.angular.z))
        hastighet = int(fart * MAX_PWM)
        vinkel = int(svang * MAX_VINKEL)
        self.mqtt.publish(self.cmd_topic, f'{hastighet},{vinkel}')


def main():
    rclpy.init()
    node = BilBrygga()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    node.mqtt.publish(node.cmd_topic, '0,0')     # stanna bilen när bryggan stängs
    node.mqtt.loop_stop()
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Förklaring av de viktigaste raderna:

| Rad | Vad den gör |
|---|---|
| `mqtt.Client()` + `loop_start()` | Startar MQTT-klienten i en egen tråd så att den inte blockerar `rclpy.spin`. |
| `on_message = self.fran_bilen` | Anropas varje gång ett MQTT-meddelande kommer — precis som en ROS2-callback. |
| `fart * MAX_PWM` | `Twist` i ROS2 är "−1…1 = full fart bakåt/framåt". Firmware vill ha PWM. Bryggan gör omräkningen så att dina noder slipper veta något om PWM. |
| `Range` | Standardmeddelandet i ROS2 för avståndssensorer. Kolla fälten med `ros2 interface show sensor_msgs/msg/Range`. |

### Registrera, bygg, kör

Lägg till i `setup.py` under `console_scripts`:

```python
'bil_brygga = min_turtle.bil_brygga:main',
```

```bash
cd ~/ros2_ws
colcon build --packages-select min_turtle
source install/setup.bash
ros2 run min_turtle bil_brygga --ros-args -p namn:=24lucsil
```

(Lägg till `-p anvandare:=... -p losenord:=...` om din firmware loggar in på brokern, och `-p broker:=maqiatto.com` om ni använder Maqiatto.)

## Del 4: Kör bilen från ROS2

Nu finns bilen i ROS2. Kolla att topics dyker upp:

```bash
ros2 topic list
ros2 topic echo /ultraljud
ros2 topic hz /ultraljud          # ska ligga runt 10 Hz
```

Styr med tangentbordet. `teleop_twist_keyboard` är ROS2:s standard-teleop och publicerar på `/cmd_vel` — samma topic som `turtle_teleop_key` fast utan `/turtle1/`-prefixet:

```bash
sudo apt install -y ros-$ROS_DISTRO-teleop-twist-keyboard
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -p repeat_rate:=10.0
```

> `repeat_rate:=10.0` gör att teleop skickar senaste kommandot 10 ggr/s även när du inte trycker på något. Utan det stannar bilen efter 500 ms — dödmansgreppet igen.

Tangenter: `i` framåt, `,` bakåt, `j`/`l` sväng, `k` stopp, `q`/`z` ökar/minskar farten. Börja med farten nerskruvad (`z` några gånger) — bilen är snabbare än sköldpaddan.

Du kan också skicka ett enstaka kommando precis som i labb 3:

```bash
ros2 topic pub --rate 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.4}, angular: {z: 0.5}}"
```

## Uppgifter

### Uppgift 1 — Dödmansgreppet

Kör bilen framåt med teleop (på lådan). Tryck `Ctrl+C` i **bryggans** terminal medan bilen kör. Hur lång tid tar det innan hjulen stannar? Gör om testet med `DODMANS_MS = 2000` i firmware. Beskriv i 2–3 meningar varför dödmansgreppet ligger i firmware och inte i bryggan eller i teleop.

### Uppgift 2 — Kalibrera riktningarna

ROS2 har en fast konvention: `linear.x > 0` är framåt och `angular.z > 0` är sväng åt **vänster** (moturs sett uppifrån). Kontrollera med `ros2 topic pub` att din bil följer konventionen. Om inte: byt tecken i `korMotor()` eller `styr()` i firmware — **inte** i bryggan eller i dina noder. Förklara varför det är rätt ställe att fixa det på.

Justera också `SERVO_CENTER` så att bilen går rakt när `angular.z = 0`.

### Uppgift 3 — Dödzon

Hitta det lägsta `hastighet`-värdet som får bilen att rulla på golvet (prova `"200,0"`, `"300,0"` … med `mosquitto_pub`). Under det värdet surrar motorn utan att bilen rör sig. Lägg till en **dödzon** i bryggan: om `0 < |hastighet| < MIN_PWM`, skicka `MIN_PWM` med rätt tecken istället. Motivera varför det hör hemma i bryggan.

### Uppgift 4 — Avståndslogg

Skriv `avstand_logg.py`: en subscriber på `/ultraljud` som loggar avståndet i **cm** var tionde meddelande (jämför labb 6, uppgift 1). Kör den samtidigt som bryggan och `ros2 node info /bil_brygga`. Skärmdump som visar bryggans publishers och subscribers.

### Uppgift 5 — Ljussensorn (utmaning, behövs i labb 10)

Koppla en **ljussensor** (LDR + 10 kΩ som spänningsdelare) till `A0`, riktad **nedåt mot golvet**, gärna med en vit LED bredvid som lyser på papperet. Lägg till i firmware:

```cpp
// i loop(), i samma block som avståndet:
client.publish(String(MQTT_NAMN) + "/ljus", String(analogRead(A0)));   // 0-1023
```

Bryggan prenumererar på `<namn>/ljus` och publicerar värdet som `std_msgs/msg/Int32` på `/ljus` (samma mönster som `fran_bilen`, men utan omräkning). Håll ett vitt och ett svart papper under sensorn och se skillnaden med `ros2 topic echo /ljus` — skriv upp de två värdena, du behöver tröskeln i labb 10.

## Inlämning

1. `ros_bil.ino` (din firmware) och `bil_brygga.py`.
2. Skärmdump med tre terminaler: bryggan, `ros2 topic echo /ultraljud` och teleop.
3. Svar på uppgift 1 och 2.
4. Bryggan med dödzon (uppgift 3) + värdet du hittade.
5. `avstand_logg.py` + skärmdump från uppgift 4.
6. Visa bilen köra via teleop för någon av lärarna.

## Vanliga problem

**`mosquitto_sub` visar inget** — kolla Serial Monitor: står det `MQTT ansluten`? Har du samma prefix i `MQTT_NAMN` och i kommandot? Använder firmware inloggning måste `mosquitto_sub` ha `-u` och `-P`.

**Hjulen snurrar en halv sekund och stannar** — det är rätt! Dödmansgreppet. Skicka kommandon oftare (teleop med `repeat_rate`, eller `ros2 topic pub --rate 10`).

**`ModuleNotFoundError: No module named 'paho'`** — du installerade inte `python3-paho-mqtt`, eller så installerade du det i en annan Python än den ROS2 använder. Använd `apt`, inte `pip`.

**`ValueError: Unsupported callback API version`** — du har paho-mqtt 2.x. Byt `mqtt.Client()` mot `mqtt.Client(mqtt.CallbackAPIVersion.VERSION1)`.

**`ModuleNotFoundError: No module named 'sensor_msgs'`** — kör `source /opt/ros/$ROS_DISTRO/setup.bash` och bygg om.

**Bilen rycker eller reagerar sent** — kolla `ros2 topic hz /cmd_vel`. Publicerar du mer än ~20 ggr/s? Sänk takten; molnbrokern behöver inte fler meddelanden än så.

**Motorn går men servot rör sig inte (eller tvärtom)** — dubbelkolla pinnarna mot din gamla firmware. `D4` på många kort är även den inbyggda LED:en, vilket normalt är okej för servo.

**Två bilar reagerar på mina kommandon** — någon annan i klassen har samma `MQTT_NAMN`. Prefixet måste vara unikt.

## Användbara kommandon

```bash
mosquitto_sub -h <broker> -t <topic>              # lyssna på MQTT
mosquitto_pub -h <broker> -t <topic> -m "<text>"  # skicka MQTT
ros2 run min_turtle bil_brygga --ros-args -p namn:=<prefix>
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -p repeat_rate:=10.0
ros2 topic hz /ultraljud                          # meddelanden per sekund
ros2 interface show sensor_msgs/msg/Range
```
