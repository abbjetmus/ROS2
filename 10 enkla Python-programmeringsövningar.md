# Python

## Innehåll

1. [Variabler](#1-variabler)
2. [Datatyper](#2-datatyper)
3. [Operatorer](#3-operatorer)
4. [Jämförelser](#4-jämförelser)
5. [If-satser](#5-if-satser)
6. [Loopar](#6-loopar)
7. [Funktioner](#7-funktioner)
8. [Strängar](#8-strängar)
9. [Listor](#9-listor)
10. [Blandade övningar](#10-blandade-övningar)

## Introduktion

Här är 10 enkla Python-övningar på ungefär samma nivå som JavaScript-materialet du visade. Övningarna följer en tydlig progression och fokuserar på grundläggande programmeringskoncept.

### Kursinformation

- **Mål:** Lära dig grunderna i Python-programmering
- **Metod:** Kombination av teori och praktiska övningar
- **Resurser:** Text och hands-on kodning
- **Tips:** Testa gärna egna lösningar och experimentera med koden!

---

## 1. Variabler

Skapa variabler som innehåller ditt namn, din ålder och din favoritfärg. Skriv sedan ut värdena.

**Exempel:**
```text
Namn: Anna
Ålder: 20
Favoritfärg: blå
```

**Tips:** Använd variabler och `print()`.

## 1. Hälsa på användaren

Skapa ett program som frågar efter användarens namn och sedan skriver ut en hälsning.

**Exempel:**
```text
Vad heter du? Anna
Hej Anna!
```

---

## 2. Datatyper

Skapa variabler med datatyperna `str`, `int`, `float` och `bool`. Skriv ut värdet och datatypen för varje variabel.

**Exempel:**
```text
Kalle <class 'str'>
20 <class 'int'>
3.14 <class 'float'>
True <class 'bool'>
```

**Tips:** Använd `type()`.

---

## 3. Operatorer

Be användaren skriva in två tal. Räkna ut summan, differensen, produkten och kvoten.

**Exempel:**
```text
Tal 1: 10
Tal 2: 5
Summa: 15
Differens: 5
Produkt: 50
Kvot: 2.0
```

**Tips:** Använd `input()` och omvandla värdena med `int()` eller `float()`.

## 4. Jämförelser

Be användaren skriva in två tal och skriv ut om det första talet är större, mindre eller lika stort som det andra.

**Exempel:**
```text
Tal 1: 12
Tal 2: 8
12 är större än 8.
```

**Tips:** Använd jämförelseoperatorerna `>`, `<` och `==`.

---

## 5. If-satser

Be användaren skriva in sin ålder. Om personen är 18 år eller äldre ska programmet skriva att personen är myndig. Annars ska programmet skriva att personen inte är myndig.

**Exempel:**
```text
Ålder: 20
Du är myndig.
```

**Tips:** Använd `if` och `else`.

## 6. Loopar

Skriv ett program som skriver ut talen från 1 till 10 med en `for`-loop. Skapa sedan en variant som räknar från 10 ner till 1 med en `while`-loop.

**Förväntat resultat:**
```text
1
2
3
4
5
6
7
8
9
10
```

**Tips:** Använd en `for`-loop och `range()`.

## 7. Funktioner

Skapa en funktion som tar emot två tal och returnerar summan av dem. Anropa sedan funktionen med olika värden.

**Exempel:**
```text
Resultat: 8
```

**Tips:** Använd `def`, parametrar och `return`.

---

## 8. Strängar

Be användaren skriva in en text. Skriv sedan ut texten med versaler, antalet tecken och textens första tecken.

**Exempel:**
```text
Skriv en text: Hej världen
HEJ VÄRLDEN
Antal tecken: 11
Första tecknet: H
```

**Tips:** Använd metoder som `.upper()` och funktionen `len()`.

---

## 9. Listor

Skapa en lista med fem favoritfilmer eller favoritspel. Skriv ut hela listan, det första elementet och det sista elementet. Lägg sedan till ett nytt element i listan.

**Exempel:**
```text
['Minecraft', 'Zelda', 'Mario', 'Portal', 'Tetris']
Första: Minecraft
Sista: Tetris
```

**Tips:** Använd indexering och `.append()`.

---

## 10. Blandade övningar

Skapa ett program som frågar användaren hur många poäng den fick på ett test och skriver ut ett betyg.

Använd följande regler:

| Poäng | Betyg |
|---:|:---|
| 90–100 | A |
| 80–89 | B |
| 70–79 | C |
| 60–69 | D |
| 50–59 | E |
| 0–49 | F |

**Exempel:**

```text
Hur många poäng fick du? 83
Ditt betyg är B.
```

**Extra utmaning:** Kontrollera också att användaren inte skriver in ett tal under 0 eller över 100.

Försök därefter kombinera flera av koncepten ovan: använd en funktion för att räkna ut betyget och en loop för att låta användaren göra flera tester.