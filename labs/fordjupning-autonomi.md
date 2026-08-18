# Fördjupning — autonom robotik med turtlesim

**Frivillig bonus efter labb 8 eller efter godkänd grundversion av robotmoppen**

Den här delen är för elever som vill gå längre med autonom logik. Uppgifterna tränar mer programmering, geometri och reglering än grundlabbarna och är därför **inte obligatoriska**.

Börja först när manuell styrning, sensor och säkerhetsstopp fungerar stabilt. Testa alltid nya beteenden i turtlesim innan de eventuellt används på den fysiska robotmoppen. Fysisk testning görs endast enligt lärarens instruktioner.

## Vad är skillnaden mot grundlabbarna?

I grundspåret ligger fokus på ROS2:

```text
noder → topics → messages → publisher/subscriber → parameter → launch
```

Här ligger fokus mer på beteende:

```text
sensorvärde → beräkning → beslut → styrning
```

Du bör vara bekväm med labb 1–7 innan du börjar.

---

## Fördjupning 1 — Vänd istället för att stanna

Utgå från `stoppare.py` i labb 6.

Målet är att roboten ska:

1. köra framåt,
2. upptäcka att den är nära en kant,
3. rotera en stund,
4. fortsätta framåt igen.

Ett enkelt sätt är att använda ett tillstånd:

```python
self.vander = False
self.vand_ticks = 0
```

När kanten upptäcks:

```python
self.vander = True
self.vand_ticks = 0
```

När roboten vänder:

```python
cmd.linear.x = 0.0
cmd.angular.z = 1.5
self.vand_ticks += 1
```

Efter ett bestämt antal tickar går du tillbaka till framåtkörning.

### Utmaning

Kan du göra vändtiden till en parameter?

---

## Fördjupning 2 — Vägg-undvikare med tillstånd

Bygg en nod med två tydliga states:

```text
KOR_FRAMAT
VANDA
```

Exempel:

```python
KOR_FRAMAT = 'kor_framat'
VANDA = 'vanda'
```

Systemet ska byta state när roboten kommer nära en kant.

### Viktigt

Byt **inte** tillbaka från `VANDA` enbart för att `x`/`y` inte längre är nära väggen om roboten roterar på plats — positionen ändras ju inte när den bara roterar.

Använd i stället exempelvis:

- en timer/räknare för hur länge vändningen pågår,
- eller robotens `theta` för att avgöra när vändningen är klar.

Detta är ett bra tillfälle att fundera på skillnaden mellan **position** och **riktning**.

---

## Fördjupning 3 — Smart vändning mot mitten

Om du vill använda robotens riktning kan du beräkna en målvinkel mot mitten av arenan.

```python
import math

mal_theta = math.atan2(
    5.5 - msg.y,
    5.5 - msg.x
)
```

Beräkna sedan vinkelfelet:

```python
vinkelfel = mal_theta - msg.theta
vinkelfel = math.atan2(
    math.sin(vinkelfel),
    math.cos(vinkelfel)
)
```

En enkel styrregel kan vara:

```python
cmd.angular.z = 1.5 * vinkelfel
```

### Frågor

1. Varför behöver vinkelfelet normaliseras?
2. Vad händer om förstärkningen `1.5` blir mycket större?
3. Vad händer om den blir mycket mindre?

---

## Fördjupning 4 — Måljakt

Skapa en nod `jaga_mal.py`.

Välj ett mål:

```python
self.mal_x = 8.0
self.mal_y = 3.0
```

I `pose_callback`:

1. räkna avstånd till målet,
2. räkna riktning mot målet,
3. jämför med `theta`,
4. styr rotationen efter vinkelfelet,
5. styr farten efter avståndet,
6. stanna när roboten är nära målet.

Exempel:

```python
dx = self.mal_x - msg.x
dy = self.mal_y - msg.y

avstand = math.sqrt(dx * dx + dy * dy)
mal_theta = math.atan2(dy, dx)

vinkelfel = mal_theta - msg.theta
vinkelfel = math.atan2(
    math.sin(vinkelfel),
    math.cos(vinkelfel)
)
```

Börja med enkla värden:

```python
cmd.linear.x = 0.8 * avstand
cmd.angular.z = 2.0 * vinkelfel
```

Begränsa gärna maxhastigheten så att roboten inte blir svår att kontrollera.

---

## Fördjupning 5 — Flera autonoma sköldpaddor

Gör din styrnod återanvändbar genom en parameter:

```python
self.declare_parameter('turtle', 'turtle1')
turtle = self.get_parameter('turtle').value
```

Använd sedan:

```python
f'/{turtle}/cmd_vel'
f'/{turtle}/pose'
```

Starta samma nod två gånger:

```bash
ros2 run min_turtle min_nod --ros-args -p turtle:=turtle1
ros2 run min_turtle min_nod --ros-args -p turtle:=turtle2
```

### Utmaning

Skapa en launch-fil som startar båda instanserna.

---

## Fördjupning 6 — Fastnat-detektor

Spara robotens position med jämna mellanrum.

Om den nästan inte har flyttat sig på några sekunder trots att den försöker köra framåt kan du anta att den har fastnat.

Ett möjligt kriterium:

```text
förflyttning < 0.5 enheter under 3 sekunder
```

När det händer kan du:

- stoppa,
- rotera längre än normalt,
- eller välja en slumpmässig rotationsriktning.

### Reflektion

Hur skulle en riktig mobil robot kunna upptäcka samma problem? Exempelvis med hjulodometri, lidar, motorström eller kamera.

---

## Frivilligt slutmål

Kombinera flera idéer till en autonom robot som:

- håller sig inne i arenan,
- kan ta sig mot ett mål,
- inte fastnar lika lätt,
- och kan startas med en launch-fil.

Det är ett **fördjupningsprojekt**, inte ett krav för grundkursen.
