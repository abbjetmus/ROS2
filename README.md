# ROS2 med turtlesim — labbserie

Labbar för modulen *Robotik och ROS2* på Hitachigymnasiet. Eleverna bygger upp kunskap om ROS2 stegvis med hjälp av turtlesim, från att starta simulatorn till att skriva egna autonoma noder.

**Lärare:** Ahmad Alali, Jeton Mustini, Sofie Ahlberg

## Miljö

| Del | Verktyg |
|---|---|
| Operativsystem | Windows 11 |
| Linuxmiljö | WSL2 med Ubuntu |
| ROS2-version | Jazzy (Ubuntu 24.04) eller Humble (Ubuntu 22.04) |
| Editor | Visual Studio Code med WSL-extension |
| Simulator | turtlesim |
| Språk | Python (rclpy) |

All ROS2-kod ska ligga i Linuxmiljön, t.ex. `/home/<användarnamn>/ros2_ws`, **inte** i Windows Dokument-mapp.

## Installation och setup

Gör detta **en gång** innan labb 1. Följ stegen i ordning.

### Steg 1 — Installera WSL2 + Ubuntu

Följ videon: **[Installera WSL (video 1)](https://www.youtube.com/watch?v=C6eQ6VwTpxk)**

Kort sammanfattning: öppna **PowerShell som administratör** i Windows och kör `wsl --install -d Ubuntu`. Starta om datorn, starta sedan **Ubuntu** från Start-menyn och skapa användarnamn och lösenord.

### Steg 2 — Koppla till GitHub

Följ videon: **[Koppla till GitHub (video 2)](https://www.youtube.com/watch?v=7FKi-waQuMM)**

### Steg 3 — Installera VS Code + WSL-extension

1. Installera [Visual Studio Code](https://code.visualstudio.com/) i Windows.
2. Installera tillägget **WSL** (utgivare: Microsoft) i VS Code.
3. Öppna VS Code från Ubuntu-terminalen med `code .` så att det står **`WSL: Ubuntu`** längst ner till vänster.

### Steg 4 — Installera ROS2

Följ instruktionerna i den officiella guiden för **ROS2 Humble** (Ubuntu 22.04):
[docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html)

Följ guiden ända **fram till och med**:

```bash
source /opt/ros/humble/setup.bash
```

När det kommandot körts utan fel är du **klar för labb 1**.

> **Tips:** lägg till raden i din `.bashrc` så slipper du köra den i varje ny terminal:
> ```bash
> echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
> source ~/.bashrc
> ```

### Felsökning

| Problem | Lösning |
|---|---|
| `ros2: command not found` | Kör `source /opt/ros/humble/setup.bash` igen (se tipset i steg 3). |
| Turtlesim-fönstret öppnas inte | Kör `wsl --shutdown` i PowerShell, starta Ubuntu igen. |
| VS Code visar inte `WSL: Ubuntu` | Stäng VS Code, kör `code .` från Ubuntu-terminalen igen. |

## Labbar

| Nr | Titel | Innehåll |
|---|---|---|
| [Labb 1](labs/labb-1-turtlesim.md) | Visa sköldpaddan | Starta turtlesim, styra med tangentbordet, förstå noder |
| [Labb 2](labs/labb-2-topics.md) | Vad skickas mellan noderna? | Topics, messages, `cmd_vel` och `pose` |
| [Labb 3](labs/labb-3-publicera.md) | Publicera egna kommandon | Skicka `Twist`-meddelanden från terminalen, services |
| [Labb 4](labs/labb-4-workspace.md) | Workspace och package | Skapa `ros2_ws`, bygga med `colcon`, eget package |
| [Labb 5](labs/labb-5-publisher.md) | Publisher-nod i Python | Skriva en egen nod som styr sköldpaddan |
| [Labb 6](labs/labb-6-subscriber.md) | Subscriber-nod i Python | Läsa `/turtle1/pose`, reagera på data |
| [Labb 7](labs/labb-7-vagg-undvikare.md) | Autonom vägg-undvikare | Kombinera publisher och subscriber till ett beteende |
| [Labb 8](labs/labb-8-slutprojekt.md) | Slutprojekt: autonom sköldpadda | Eget projekt som lämnas in |

## Inlämning

Varje labb har en kort inlämning i slutet (svar på frågor, kodfil, eller skärmdump). Skapa ett repo på länken och ladda upp varje labb i en egen markdown fil, tex lab1.md. [https://classroom-app.cloud.mustini.com/join/885f154c7cf1c7c5693b](https://classroom-app.cloud.mustini.com/join/885f154c7cf1c7c5693b)

## Förkunskaper

- Grundläggande programmering (variabler, loopar, funktioner)
- Vana vid att använda terminalen är en fördel men inget krav

## Resurser

- **Videoserie (spellista):** [ROS2 turtlesim-serie på YouTube](https://www.youtube.com/playlist?list=PLSK7NtBWwmpTS_YVfjeN3ZzIxItI1P_Sr) — komplement till labbarna.
- [ROS officiella hemsida](https://www.ros.org/)
- **Installation:** [Installera ROS2 Humble på Ubuntu (Debian-paket)](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html)
- [Officiell ROS2-dokumentation (Jazzy)](https://docs.ros.org/en/jazzy/)
- [ROS2 turtlesim-tutorial](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Introducing-Turtlesim/Introducing-Turtlesim.html)
