# Labb 8: Från turtlesim till robotmoppen

**Testa gränssnitten utan hårdvara och planera den fysiska integrationen**

**Förkunskaper:** Labb 1–7

## Syfte

I labb 7 fungerade turtlesim som robot och `sim_safety_sensor` som sensor. Nu ska du visa att säkerhetsnoden fungerar även när simulatorn tas bort.

Du ska inte skriva en ny avancerad algoritm. I stället ska du:

1. testa varje gränssnitt som en svart låda,
2. bestämma vilka noder den fysiska robotmoppen behöver,
3. dokumentera topics och ansvar,
4. och skapa en tydlig integrationsplan för gruppen.

Det här är övergången mellan **Smarta system** och **Teknik 2**.

## Mål

Efter labben ska du kunna:

- testa en ROS2-nod utan att all hårdvara finns,
- använda terminalen för att simulera sensor, styrning och motor,
- skriva ett enkelt systemkontrakt,
- förklara vad som ska återanvändas och vad som ska bytas,
- och planera en säker stegvis integration.

## Arkitekturen på den fysiska robotmoppen

```text
[controller_node]
        │
        │ /robotmopp/cmd_vel_raw
        ▼
   [safety_node] ◄──── /robotmopp/safety_stop ──── [sensor_node]
        │
        │ /robotmopp/cmd_vel
        ▼
    [motor_node]
        │
        ▼
  motordrivare + motorer
```

### Vad återanvänds?

`safety_node.py` från labb 7 ska kunna återanvändas.

### Vad byts?

- `sim_safety_sensor` ersätts av en nod som läser en riktig sensor.
- turtlesim ersätts av en nod som skickar data till motordrivaren.
- teleop kan först vara tangentbord och senare bytas mot joystick eller annan manuell styrning.

## Del 1: Starta bara säkerhetsnoden

```bash
cd ~/ros2_ws
source install/setup.bash
ros2 run min_turtle safety_node
```

Öppna tre ytterligare terminaler.

### Terminal A — låtsas vara motorn

```bash
ros2 topic echo /robotmopp/cmd_vel
```

### Terminal B — låtsas vara sensorn

Börja med fri väg:

```bash
ros2 topic pub --rate 2 /robotmopp/safety_stop \
  std_msgs/msg/Bool "{data: false}"
```

### Terminal C — låtsas vara styrningen

```bash
ros2 topic pub --rate 2 /robotmopp/cmd_vel_raw \
  geometry_msgs/msg/Twist \
  "{linear: {x: 1.0}, angular: {z: 0.0}}"
```

I Terminal A ska kommandot skickas vidare.

Stoppa sensor-publishern med `Ctrl+C` och starta den med risk:

```bash
ros2 topic pub --rate 2 /robotmopp/safety_stop \
  std_msgs/msg/Bool "{data: true}"
```

Nu ska framåthastigheten i Terminal A bli `0.0`.

## Del 2: Genomför ett gränssnittstest

Testa minst följande kombinationer:

| Nr | `safety_stop` | `linear.x` | `angular.z` | Förväntat output |
|---|---:|---:|---:|---|
| 1 | `false` | `1.0` | `0.0` | framåt tillåts |
| 2 | `true` | `1.0` | `0.0` | framåt blockeras |
| 3 | `true` | `-1.0` | `0.0` | backning tillåts |
| 4 | `true` | `0.0` | `1.0` | rotation tillåts |
| 5 | ingen aktuell sensorsignal | `1.0` | `0.0` | framåt blockeras |

För test 5 kan du antingen starta om `safety_node` utan sensor-publisher, eller stoppa sensor-publishern och vänta längre än parametern `sensor_timeout_sec` innan du skickar ett nytt framåtkommando.

Det här är ett **black-box-test**: du testar vad noden gör utifrån input och output, utan att behöva ändra koden.

## Del 3: Bestäm gruppens systemkontrakt

Använd [robotmoppens systemkontrakt](../robotmopp/systemarkitektur.md) som gemensam standard.

Fyll i en tabell för er grupp:

| Nod | Input | Output | Vem ansvarar? | Testas hur? |
|---|---|---|---|---|
| `controller_node` | tangentbord/joystick | `/robotmopp/cmd_vel_raw` | ... | ... |
| `sensor_node` | fysisk sensor | `/robotmopp/safety_stop` | ... | ... |
| `safety_node` | rått kommando + säkerhet | `/robotmopp/cmd_vel` | ... | ... |
| `motor_node` | `/robotmopp/cmd_vel` | motordrivare | ... | ... |

Topicsens namn och message-typer ska vara samma för alla i gruppen. Då kan varje del utvecklas och testas separat.

## Del 4: Rita två arkitekturer

Rita:

### A — Simulatorversionen

```text
teleop → safety_node → turtlesim
            ▲
            │
   sim_safety_sensor
```

### B — Fysiska robotmoppen

```text
controller_node → safety_node → motor_node
                       ▲
                       │
                  sensor_node
```

Markera tydligt:

- vad som är oförändrat,
- vad som byts,
- vilka topics som binder ihop delarna.

## Del 5: Planera integrationen stegvis

Planera i följande ordning:

1. **Mjukvarutest utan hårdvara** — terminalverktyg simulerar alla inputs och outputs.
2. **Motorbänk** — hjulen lyfts från golvet och `motor_node` testas separat.
3. **Manuell körning** — controller → safety → motor, utan moppmekanism.
4. **Sensortest** — sensorn publicerar säkerhetssignal, motorn är fortfarande frånkopplad eller hjulen upplyfta.
5. **Säkerhetsintegration** — verifiera att framåt stoppas men backning/rotation fungerar.
6. **Moppmekanism** — konstruktionen monteras och testas separat.
7. **Helhetstest** — roboten körs långsamt på avgränsad yta.

Den ordningen minskar felsökning och gör att gruppen vet vilken del som orsakar ett fel.

## Del 6: Koppling mellan kurserna

### Teknik 2 ansvarar främst för

- krav och konstruktion,
- CAD och ritningar,
- chassi, infästningar och tillverkning,
- val och placering av komponenter,
- moppmekanism,
- mekaniska tester och förbättringar.

### Smarta system ansvarar främst för

- ROS2-package och launch,
- noder och topics,
- controller-, sensor-, safety- och motorgränssnitt,
- systemtest och loggning,
- felsökning av dataflödet.

### Gemensamt ansvar

- systemarkitektur,
- säkerhet och riskanalys,
- kabeldragning och komponentplacering,
- integration,
- testprotokoll,
- slutdemonstration och reflektion.

---

## GRUND — obligatoriskt

### Uppgift 1 — Black-box-test

Genomför de fem testerna i Del 2 och dokumentera faktiskt output.

### Uppgift 2 — Systemkontrakt

Fyll i nodtabellen med inputs, outputs, ansvarig och testmetod.

### Uppgift 3 — Två arkitekturdiagram

Rita simulatorversionen och den fysiska versionen. Markera vad som återanvänds.

### Uppgift 4 — Integrationsplan

Skriv gruppens ordning för minst fem teststeg. Varje steg ska ha ett tydligt godkänt-kriterium.

### Uppgift 5 — Kort muntlig kontroll

Varje elev ska kunna svara på:

1. Varför får `motor_node` aldrig lyssna direkt på `/robotmopp/cmd_vel_raw`?
2. Vilken nod kan testas utan att sensor och motor finns?
3. Vad händer om sensorn slutar skicka data?
4. Vilken del tillhör främst Teknik 2 och vilken tillhör främst Smarta system?

## FÖRDJUPNING — frivilligt

Gör bara detta efter att grundtesterna är godkända:

- lägg till ett separat nödstopp,
- publicera rått avstånd för övervakning,
- lägg till maxhastighet som parameter,
- eller skapa en enkel statuspanel i terminalen.

Autonom väggundvikning och städmönster finns i [Fördjupning — autonom robotik](fordjupning-autonomi.md), men är inte en del av grundkraven.

## Inlämning

1. Ifyllt black-box-test.
2. Gruppens systemkontrakt.
3. Två arkitekturdiagram.
4. Integrationsplan med godkänt-kriterier.
5. Kort fördelning av ansvar mellan kurserna och gruppmedlemmarna.

Efter godkänd labb fortsätter gruppen med [Robotmopp — elevprojekt](../robotmopp/README.md).
