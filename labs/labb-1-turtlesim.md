# Labb 1: Visa sköldpaddan

**Introduktion till ROS2 med turtlesim, WSL och VS Code**

**Nivå:** Nybörjare
**Miljö:** Windows 11, WSL2, Ubuntu, ROS2, VS Code

## Syfte

Du ska få din första praktiska kontakt med ROS2. Du startar en robotsimulator (turtlesim), styr en sköldpadda med tangentbordet, och undersöker hur ett ROS2-system består av flera program som kommunicerar via meddelanden.

## Mål

Efter labben ska du kunna:

- Starta en ROS2-simulator från terminalen.
- Förklara vad en **node** är och varför ett robotsystem delas upp i flera noder.
- Se vilka noder som körs med `ros2 node list`.
- Beskriva hur `turtle_teleop_key` och `turtlesim_node` kommunicerar.

## Bakgrund

**ROS2** (*Robot Operating System 2*) är inte ett operativsystem utan ett ramverk för robotprogramvara. Det hjälper utvecklare att bygga system där flera program — kallade **noder** — körs samtidigt och kommunicerar med varandra.

Ett verkligt robotsystem kan bestå av:

```
/kamera ──► /objektdetektion ──► /navigation ──► /motorstyrning
                                       ▲
                                       │
                                    /lidar
```

Varje ruta är en nod. I den här labben kör vi bara två noder:

| Nod | Vad den gör |
|---|---|
| `/turtlesim` | Visar och simulerar sköldpaddan. |
| `/teleop_turtle` | Läser piltangenterna och skickar styrkommandon. |

ROS2 körs bäst i Linux, så vi använder **WSL2** med Ubuntu inuti Windows. **VS Code** öppnas från WSL-terminalen för att hamna i rätt miljö.

## Del 1: Kontrollera miljön

Öppna **Ubuntu** från Start-menyn i Windows. Du bör nu se en terminal.

```bash
pwd
ros2 --version
```

`pwd` ska visa något i stil med `/home/dittnamn`. Om `ros2 --version` ger felmeddelande behöver du aktivera miljön:

```bash
# ROS2 Jazzy
source /opt/ros/jazzy/setup.bash

# eller ROS2 Humble
source /opt/ros/humble/setup.bash
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

Öppna en terminal (i VS Code: `Terminal → New Terminal`) och kör:

```bash
ros2 run turtlesim turtlesim_node
```

Ett blått fönster med en sköldpadda ska öppnas.

## Del 4: Styr sköldpaddan

Lämna turtlesim-fönstret öppet. Öppna en **ny terminal** (klicka på `+` i VS Code) och kör:

```bash
ros2 run turtlesim turtle_teleop_key
```

Klicka i den nya terminalen så att den är aktiv. Använd piltangenterna för att styra sköldpaddan.

## Del 5: Undersök systemet

Lämna båda programmen igång. Öppna en **tredje terminal**:

```bash
ros2 node list
```

Du ska se:

```
/teleop_turtle
/turtlesim
```

Studera varje nod:

```bash
ros2 node info /turtlesim
ros2 node info /teleop_turtle
```

Leta efter raderna under `Subscribers:`, `Publishers:` och `Services:`. Det är så du senare ser hur noder är kopplade.

## Uppgifter

Lös följande uppgifter och dokumentera kort i din inlämning (1–2 meningar per uppgift, plus skärmdumpar där det är relevant).

### Uppgift 1 — Rita en fyrkant

Använd piltangenterna och rita en så fyrkantig figur du kan. Ta en skärmdump av resultatet.

### Uppgift 2 — Rita ditt initial

Rita första bokstaven i ditt förnamn så tydligt du kan med sköldpaddan. Skärmdump.

### Uppgift 3 — Räkna kopplingar

Kör `ros2 node info /turtlesim`. Räkna hur många **subscribers** och hur många **publishers** noden har. Skriv ner siffrorna.

### Uppgift 4 — Hitta den gemensamma kanalen

Jämför utdatat från `ros2 node info /turtlesim` och `ros2 node info /teleop_turtle`. Vilket **topic** finns hos båda noderna? Vilken av noderna **publicerar** på det och vilken **subscriberar**?

### Uppgift 5 — Experiment med att stänga noder

1. Gå till terminalen där `turtle_teleop_key` körs och tryck `Ctrl+C`. Försök sedan styra sköldpaddan. Vad händer?
2. Starta `turtle_teleop_key` igen. Försök sedan stänga `turtlesim_node` med `Ctrl+C`. Vad händer med fönstret? Vad händer om du kör `ros2 node list` nu?

Förklara med egna ord vad du lärde dig av experimenten.

### Uppgift 6 — Två sköldpaddor (utmaning)

I en fjärde terminal, kör:

```bash
ros2 service call /spawn turtlesim/srv/Spawn "{x: 2.0, y: 2.0, theta: 0.0, name: 'turtle2'}"
```

Kör `ros2 node list` igen och `ros2 topic list`. Vad är nytt? Kan du styra `turtle2` med samma `turtle_teleop_key`? Varför / varför inte?

## Inlämning

Lämna in ett dokument med:

1. Skärmdumpar från uppgift 1 och 2.
2. Svar på uppgift 3, 4, 5 och 6.
3. Tre meningar där du beskriver vad en **node** är, vad ett **topic** är, och varför ROS2 delar upp ett system i flera program.

## Felsökning

**`ros2: command not found`** — Aktivera miljön: `source /opt/ros/jazzy/setup.bash` (eller `humble`).

**Sköldpaddan rör sig inte** — Är `turtle_teleop_key`-terminalen aktiv (klickad i)? Använder du piltangenterna och inte WASD?

**Turtlesim-fönstret öppnas inte** — Stäng WSL från PowerShell med `wsl --shutdown` och starta Ubuntu igen.

**VS Code visar inte `WSL: Ubuntu`** — Stäng VS Code, gå tillbaka till Ubuntu-terminalen och kör `cd ~/ros2_ws && code .`.
