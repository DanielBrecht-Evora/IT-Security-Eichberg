# Cheat Sheet — IT-Passwort-Recovery

Für das Advanced Practical IT Security Lab. Alle Befehle laufen im Linux-/Kali-Shell (bzw. WSL/Ubuntu unter Windows), sofern nicht anders vermerkt.

---

## 1. Datei-Analyse (Basis)

```bash
file datei.ext              # Dateityp bestimmen
binwalk datei.ext           # eingebettete Daten und Signaturen finden
strings datei.ext           # lesbare Zeichenketten ausgeben
strings -n 8 datei.ext      # nur Strings ab 8 Zeichen
xxd datei.ext | head        # Hex-Dump der ersten Zeilen
xxd datei.ext | less        # Hex-Dump durchblättern
ls -la                      # Dateien mit Größe und Rechten
```

---

## 2. Hash aus Datei extrahieren (*2john-Tools)

Die John-Suite bringt für viele Formate einen Extraktor mit. Die Ausgabe ist immer eine Hash-Zeile.

```bash
libreoffice2john datei.odt    > out.hash   # ODF (odt, ods, odp)
office2john      datei.docx   > out.hash   # MS Office (docx, xlsx, pptx)
7z2john          datei.7z     > out.hash   # 7-Zip
zip2john         datei.zip    > out.hash   # ZIP
rar2john         datei.rar    > out.hash   # RAR
pdf2john         datei.pdf    > out.hash   # PDF
iwork2john       datei.numbers> out.hash   # Apple iWork (numbers, pages, key)
openssl2john     datei.enc    > out.hash   # OpenSSL enc
keepass2john     datei.kdbx   > out.hash   # KeePass (kdbx < v4)
ssh2john         id_rsa       > out.hash   # SSH Private Key
```

Tools finden, falls nicht im PATH:

```bash
locate 2john                          # alle *2john auflisten
ls /usr/share/john/                   # typischer Ort unter Kali
python3 /usr/share/john/pdf2john.py   # manche sind Python-Skripte
```

Hinweis: Sehr neue Formate wie kdbx v4 oder manche OpenSSL-KDFs werden von john/hashcat oft nicht unterstützt. Dann brauchst du ein eigenes Python-Skript (siehe deine Aufgaben 6 und 8).

---

## 3. Hash für hashcat säubern

john liefert `datei:HASH:::::pfad`. hashcat will nur den reinen Hash.

```bash
# Präfix "datei:" und Suffix ":::::pfad" entfernen
sed -E 's/^[^:]+://; s/:::::[^:]+$//' out.hash > out.hashcat

# Variante wenn 4 oder mehr Doppelpunkte folgen (z. B. iwork)
sed -E 's/^[^:]+://; s/:{4,}.*$//' out.hash > out.hashcat
```

---

## 4. hashcat — Grundgerüst

```bash
hashcat -m MODUS -a ANGRIFF out.hashcat wortliste.txt
hashcat --show out.hashcat ...     # bereits geknackte Treffer anzeigen
hashcat --identify out.hashcat     # passenden -m Modus vorschlagen
hashcat --help | grep -i pdf       # Modusnummer zu einem Format suchen
```

Wichtige Optionen

```
-m   Hash-Typ (Beispiele unten)
-a   Angriffsmodus (siehe Abschnitt 5)
-o   Ausgabedatei für Treffer
--show               Treffer aus dem Potfile zeigen
--username           erste Spalte als Username ignorieren
--increment          Länge schrittweise erhöhen
--increment-min N    Mindestlänge
--increment-max N    Maximallänge
--force              Warnungen ignorieren
-O                   optimierter Kernel, schneller, begrenzte Länge
-w 3                 Workload 1 (niedrig) bis 4 (hoch)
--status             laufende Statusanzeige
--potfile-disable    Potfile ignorieren, immer neu cracken
```

Häufige Modusnummern (bei Unsicherheit `--identify` nutzen)

```
10500  PDF 1.4 bis 1.6      10700  PDF 1.7
11600  7-Zip                13600  WinZip
13400  KeePass              9600   MS Office 2013
18400  ODF 1.2              0      MD5
```

---

## 5. Angriffsmodi (-a)

```
-a 0  Straight    Wortliste, optional mit Regeln (-r)
-a 1  Combinator  zwei Wortlisten aneinanderhängen
-a 3  Mask        Brute-Force über Maske
-a 6  Hybrid      Wortliste + Maske
-a 7  Hybrid      Maske + Wortliste
```

### -a 0 Wortliste (mit Regeln)

```bash
hashcat -a0 out.hashcat rockyou.txt
hashcat -a0 out.hashcat rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### -a 1 Combinator (mit Einzelregeln je Liste)

```bash
hashcat -a1 out.hashcat links.txt rechts.txt
hashcat -a1 out.hashcat links.txt rechts.txt -j '$1' -k '^#'
# -j Regel auf linkes Wort, -k Regel auf rechtes Wort
```

### -a 3 Maske (Brute-Force)

```bash
hashcat -a3 out.hashcat '?u?l?l?l?d?d'
hashcat -a3 out.hashcat '?a?a?a?a' --increment
# eigene Zeichensätze mit -1 bis -4
hashcat -a3 -1 '$€!' out.hashcat '?l?l?1?1'
```

### -a 6 / -a 7 Hybrid

```bash
hashcat -a6 out.hashcat wortliste.txt '?d?d?d'   # Wort + 3 Ziffern
hashcat -a7 out.hashcat '?d?d' wortliste.txt     # 2 Ziffern + Wort

# Beispiel aus Aufgabe 3 (Maske + zusammengesetzte Wortliste, mit Sonderzeichen)
hashcat -a6 -1 '$€!' star.hashcat jahre_staedte.txt '?1?1?1?1?1?1' \
  --increment --increment-min 4 --increment-max 6
```

---

## 6. Masken-Platzhalter

```
?l   a-z
?u   A-Z
?d   0-9
?s   Sonderzeichen  ! " # $ % & ...
?a   ?l ?u ?d ?s zusammen
?b   0x00 bis 0xff (alle Bytes)
?1 ?2 ?3 ?4   selbst definierte Sätze (mit -1 bis -4)
```

---

## 7. Regel-Syntax (kurz)

- .rules Datei

```
:    nichts tun
l    alles klein
u    alles groß
c    erster Buchstabe groß
$X   Zeichen X anhängen
^X   Zeichen X voranstellen
r    Wort umdrehen
sXY  X durch Y ersetzen
```

Präfix voranstellen (Reihenfolge rückwärts denken, damit "Der " entsteht):

```bash
hashcat -a0 out.hashcat woerter.txt -j '^ ^r^e^D'
```

---

## 8. Wortlisten bauen und kombinieren

```bash
# rockyou entpacken (Kali)
gunzip -k /usr/share/wordlists/rockyou.txt.gz
ls /usr/share/wordlists/

# zwei Listen kreuzen und als Datei speichern
hashcat --stdout -a1 links.txt rechts.txt > kombi.txt

# Regeln auf eine Liste anwenden und speichern
hashcat --stdout -a0 woerter.txt -r regel.rule > erweitert.txt

# crunch: alle Kombinationen nach Muster
crunch 6 8 -o out.txt                # Länge 6 bis 8, Standardzeichen
crunch 4 4 abcdef -o out.txt         # nur bestimmte Zeichen

# aufräumen
sort -u liste.txt > liste_uniq.txt   # sortieren und Duplikate entfernen
wc -l liste.txt                      # Anzahl Zeilen zählen
```

---

## 9. John the Ripper (Alternative zu hashcat)

```bash
john --wordlist=rockyou.txt out.hash
john --wordlist=rockyou.txt --rules out.hash
john --incremental out.hash               # Brute-Force
john --format=raw-sha1 --wordlist=... out.hash
john --show out.hash                       # Treffer anzeigen
john --list=formats | grep -i pdf          # Format suchen
```

---

## 10. OpenSSL entschlüsseln (nach Passwort-Fund)

```bash
# klassische EVP_BytesToKey-Variante (ältere openssl-enc-Dateien)
openssl enc -d -aes-256-cbc -md sha1 \
  -in datei.enc -out klartext -pass pass:PASSWORT

# Ergebnis prüfen
file klartext
xxd klartext | head
```

---

## 11. Nützliche Helfer

```bash
hashid 'HASH'                   # Hash-Typ raten
hashcat -b                      # Benchmark
hashcat -m 0 --example-hashes   # Beispiel-Hashes je Modus
watch -n5 nvidia-smi            # GPU-Last beobachten (bei GPU-Cracking)
```

---

## 12. Typischer Ablauf im Lab (Merkkette)

1. `file` und `binwalk` zur Analyse
2. passendes `*2john` zum Hash extrahieren
3. `sed` zum Säubern für hashcat
4. Modus über `--identify` bzw. `--help | grep` bestimmen
5. Angriff wählen (Wortliste, Maske, Hybrid) und ggf. Wortliste bauen
6. bei zu neuen Formaten (kdbx v4, spezielle KDFs) eigenes Python-Skript
7. Ergebnis mit `--show` prüfen und Datei entschlüsseln

---

## 13. Hashcat und JohnTheRipper installieren
```bash
sudo apt update && sudo apt install hashcat
sudo apt update && sudo apt install john
```
