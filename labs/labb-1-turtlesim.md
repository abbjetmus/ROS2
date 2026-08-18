# Labb 1: Visa sköldpaddan

**Introduktion till ROS2 med turtlesim, WSL och VS Code**

**Nivå:** Nybörjare  
**Miljö:** Windows 11, WSL2, Ubuntu 24.04, ROS2 Jazzy, VS Code

## Syfte

Du ska få din första praktiska kontakt med ROS2. Du startar en robotsimulator (turtlesim), styr en sköldpadda med tangentbordet och undersöker hur ett ROS2-system består av flera program som kommunicerar via meddelanden.

## Mål

Efter labben ska du kunna:

- Starta en ROS2-simulator från terminalen.
- Förklara vad en **node** är och varför ett robotsystem delas upp i flera noder.
- Se vilka noder som körs med `ros2 node list`.
- Beskriva hur `turtle_teleop_key` och `turtlesim_node` kommunicerar.

## Bakgrund

**ROS2** (*Robot Operating System 2*) är inte ett operativsystem utan ett ramverk för robotprogramvara. Det hjälper utvecklare att bygga system där flera program — kallade **noder** — körs samtidigt och kommunicerar med varandra.

Robotmoppens system kommer senare att följa samma idé:

```text
/styrning ──► /sakerhet ──► /motorstyrning
                  ▲
                  │
               /sensor
```

Varje ruta är en nod. I den här labben kör vi bara två noder:

| Nod | Vad den gör |
|---|---|
| `/turtlesim` | Visar och simulerar sköldpaddan. |
| `/teleop_turtle` | Läser piltangenterna och skickar styrkommandon. |

ROS2 körs i vår Linuxmiljö i WSL2. **VS Code** öppnas från WSL-terminalen för att hamna i rätt miljö.

## Del 1: Kontrollera miljön

Öppna **Ubuntu 24.04** från Start-menyn i Windows.

```bash
pwd
echo $ROS_DISTRO
ros2 pkg list | grep turtlesim
```

`pwd` ska visa något i stil med `/home/dittnamn` och `echo $ROS_DISTRO` ska visa `jazzy`.

Om `echo $ROS_DISTRO` inte visar något, aktivera miljön:

```bash
source /opt/ros/jazzy/setup.bash
```

## Del 2: Öppna VS Code från WSL

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
code .
```

Längst ner till vänster i VS Code ska det stå **`WSL: Ubuntu`**. Om det inte gör det har VS Code öppnats i Windows — stäng och kör `code .` från Ubuntu-terminalen igen.

> **Regel:** All ROS2-kod ligger i `~/ros2_ws` i Ubuntu, aldrig i `C:\Users\...\Documents`.

## Del 3: Starta turtlesim

Öppna en terminal i VS Code och kör:

```bash
ros2 run turtlesim turtlesim_node
```

Ett blått fönster med en sköldpadda ska öppnas.

## Del 4: Styr sköldpaddan

Lämna turtlesim-fönstret öppet. Öppna en **ny terminal** och kör:

```bash
ros2 run turtlesim turtle_teleop_key
```

Klicka i den nya terminalen så att den är aktiv. Använd piltangenterna för att styra sköldpaddan.

## Del 5: Undersök systemet

Lämna båda programmen igång. Öppna en **tredje terminal**:

```bash
ros2 node list
```

Du ska se ungefär:

```text
/teleop_turtle
/turtlesim
```

Studera varje nod:

```bash
ros2 node info /turtlesim
ros2 node info /teleop_turtle
```

Leta efter raderna under `Subscribers:`, `Publishers:` och `Services:`. Det är så du senare ser hur noder är kopplade.

## Koppling till robotmoppen

På robotmoppen kommer systemet också delas upp i flera noder. En nod kan läsa styrningen, en annan kan läsa sensorn och en tredje kan styra motorerna.

```text
controller_node → safety_node → motor_node
                        ▲
                        │
                   sensor_node
```

**Kopplingsfråga:** Varför är det bättre att dela upp robotmoppen i flera noder än att lägga all funktion i ett enda stort program?

Svara med 1–2 meningar. Svaret ingår i inlämningen.

## Inlämning

Lämna in:

1. Skärmdumpar från uppgift 1 och 2.
2. Svar på uppgift 3, 4 och 5.
3. Tre meningar där du beskriver vad en **node** är, vad ett **topic** är och varför ROS2 delar upp ett system i flera program.
4. Kort svar på kopplingsfrågan om robotmoppen.

## Felsökning

**`ros2: command not found`** — kör `source /opt/ros/jazzy/setup.bash`.

**Sköldpaddan rör sig inte** — kontrollera att `turtle_teleop_key`-terminalen är aktiv och att du använder piltangenterna.

**Turtlesim-fönstret öppnas inte** — kör `wsl --shutdown` i PowerShell och starta Ubuntu igen.

**VS Code visar inte `WSL: Ubuntu`** — stäng VS Code och kör `cd ~/ros2_ws && code .` från Ubuntu-terminalen.
