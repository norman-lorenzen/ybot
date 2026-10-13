






# YBOT — ein zweibeiniger Roboter lernt gehen

Ein selbstgeschriebenes Programm in C, das einem simulierten Humanoiden das
Gehen beibringt. Rund 8.000 Zeilen, ohne Framework, ohne fremde Bibliothek
ausser der Physik-Engine.

Begonnen Ende Mai 2026, parallel zur Umschulung zum Fachinformatiker
Anwendungsentwicklung. Meine erste Zeile C stammt von April 2026.

---

## Worum es geht

Ein Roboter mit zwei Beinen steht in einer Physiksimulation. Er hat Sensoren
(Gelenkwinkel, Neigung, Bodenkontakt) und Motoren an Huefte, Knie und Fuss.
Dazwischen sitzen kleine neuronale Netze, die aus den Messwerten Motorbefehle
berechnen — fuenfhundert Mal pro Sekunde.

Am Anfang kennen diese Netze nur Zufallswerte, und der Roboter fällt sofort um.
Das Programm probiert dann tausende leicht veraenderte Varianten aus, bewertet
jede und behaelt die beste. Nach vielen Durchlaeufen entsteht daraus ein Gang.

Langfristiges Ziel ist ein Aufbau aus Carbon mit echten Servomotoren. Deshalb
sind die Netze bewusst klein und die Rechenzeit fest — es soll spaeter auf
einem Mikrocontroller laufen, nicht auf einer Grafikkarte.

---

## Technisch

**Sprache:** C, ohne Framework
**Physik:** MuJoCo
**Verfahren:** Evolutionsstrategie (kein Gradientenabstieg, kein Backpropagation)
**Parallelisierung:** POSIX-Threads, mehrere Trainingslaeufe gleichzeitig
**Umfang:** rund 8.000 Zeilen in sechs Modulen
**Plattform:** macOS

Warum kein fertiges Framework: der Reiz lag darin, jeden Teil selbst zu bauen
und zu verstehen — vom Speicherlayout der Gewichte ueber die Serialisierung bis
zur Parallelisierung. Und weil das Ergebnis auf einen Mikrocontroller soll, wo
es diese Frameworks ohnehin nicht gibt.

---

## Aufbau

Der Roboter ist nach Koerperteilen zerlegt. Jedes Teil bringt seine eigenen
Sensoren, sein eigenes Netz und seine eigene Motoransteuerung mit:

    Rumpf         Neigung, Drehraten, eigene Gelenkstellungen
    Bein links    Huefte, Knie, Fuss, Bodenkontakt
    Bein rechts   gleicher Aufbau
    Taktgeber     sagt jedem Bein, ob es gerade schwingt oder traegt
    Gewichte      Laden und Speichern, mit Versionsmigration
    Hauptteil     Trainingsschleife, Bewertung, Episodenablauf

Der Koerper selbst — Massen, Gelenkgrenzen, Motorkraft, Boden — steht in einer
Beschreibungsdatei und ist vom Programm getrennt.

Die Gewichtsdateien haben einen eigenen Kopfsatz mit Groessenangaben. Wird ein
Netz spaeter erweitert, koennen aeltere Staende weiterverwendet werden, statt
verworfen zu werden.

---

## Der Ablauf

**Je Simulationsschritt** (500 Hz): Sensoren lesen, Gangtakt bestimmen, Netze
rechnen lassen, Motorbefehle begrenzen und glaetten, Physik einen Schritt
weiter. Eine Episode endet, wenn der Roboter faellt.

**Je Kandidat:** mehrere Episoden unter festen Startbedingungen —
unterschiedliche Anschubgeschwindigkeiten und seitliche Stoesse zu
verschiedenen Zeitpunkten. Bewertet wird der schlechteste dieser Laeufe plus
ein Anteil des Mittelwerts, damit sich niemand mit einer einzigen beherrschten
Bedingung durchmogeln kann.

**Je Generation:** ein Elternstand wird vervielfaeltigt, die Kopien werden
zufaellig leicht veraendert, alle laufen parallel auf eigenen Kernen, der Beste
wird der neue Elter. Der alte Elter tritt selbst mit an und kann nur von etwas
Besserem verdraengt werden.

Rund 500.000 Simulationsschritte pro Sekunde auf zehn Kernen. In einer Stunde
Rechenzeit werden etwa fuenfzig Tage simulierter Roboterzeit durchgespielt.

---

## Das eigentliche Problem: Woran erkennt man Fortschritt?

Das war die schwierigste Frage des Projekts, und sie hat mich am meisten
gelehrt.

Die naheliegende Antwort — "die Punktzahl steigt" — ist falsch. Der Roboter
optimiert exakt das, was bewertet wird, und findet dabei zuverlaessig Wege, die
mit Gehen nichts zu tun haben:

- Er blieb regungslos stehen, weil Nicht-Umfallen Punkte gab.
- Er rutschte vorwaerts, ohne einen Fuss zu heben, weil Strecke zaehlte.
- Er trippelte mit vier Schritten pro Sekunde, weil Schritte zaehlten.
- Er hob ein Bein und setzte es an derselben Stelle wieder ab.
- Er stand eine Sekunde lang auf einem Bein, weil Einbeinstand belohnt wurde.

Jedes dieser Verhalten war unter der damaligen Bewertung die **richtige**
Loesung. Die Bewertung besteht heute aus rund fuenfzehn Teilen, und jeder
einzelne ist die Antwort auf so ein Ausweichmanoever.

Dazu kam ein zweites, unabhaengiges Mass: eine Reihe von Bedingungen, die im
Training nie vorkommen. Nur daran laesst sich ablesen, ob der Roboter wirklich
gehen gelernt hat oder sich nur an seine Trainingssituationen angepasst hat.
Mehrfach sah ein Stand nach einem Rekord aus und war an diesem Mass ein
Rueckschritt.

---

## Drei Beispiele aus der Fehlersuche

**Ein abgeklemmtes Netz.** Zwei Tage lang stieg die Bewertung, ohne dass der
Roboter besser wurde. Ursache: beim Uebergang zur naechsten Generation wurde
das Beinnetz des Siegers nicht mit uebernommen — mutiert und bewertet wurde
korrekt, nur das Ergebnis ging verloren. Die Beine waren die ganze Zeit
funktionslos, der Anstieg kam allein vom Rumpf. Gefunden ueber eine Kennzahl,
die ueber ganze Laeufe hinweg exakt null blieb.

**Ein wirkungsloser Schalter.** Ein Gelenk fuer die seitliche Balance war
auf ein Drittel seines mechanischen Wegs begrenzt. Der Parameter, der das haette
steuern sollen, griff nie — die eigentliche Begrenzung sass an einer ganz
anderen Stelle im Programm. Nach der Korrektur legte die Gehstrecke um 40 % zu,
ohne dass am Netz irgendetwas geaendert wurde.

**Eine Anzeige, die log.** Die 3D-Ansicht rechnete anfangs eine eigene zweite
Simulation nebenher. Sie driftete von der bewerteten auseinander, sodass man
beim Zuschauen etwas anderes sah als das Gemessene. Heute zeigt sie nur noch
die Stellungen an, die das Trainingsprogramm selbst herausschreibt.

---

## Stand

Der Roboter geht. Gemessen: mehrere Schritte am Stueck, dabei durchgehend
mindestens ein Fuss am Boden — nach der biomechanischen Definition also Gehen
und nicht Huepfen. Die Gehstrecke hat sich innerhalb eines Tages auf das
Zweieinhalbfache gesteigert, nachdem der oben beschriebene Gelenkfehler
gefunden war.

Ein durchgehender, beliebig langer Gang ist noch nicht erreicht. Nach einigen
Schritten faellt er.

Zu jedem gesicherten Stand gehoeren die Messwerte, die verwendete
Koerperbeschreibung und die vollstaendige Konfiguration — sonst laesst sich ein
Ergebnis spaeter weder einordnen noch nachstellen.

Der Verlauf ist in Tagesprotokollen dokumentiert: was gemessen wurde, was
funktioniert hat und was nicht, jeweils mit Zahlen.

---

## Was ich dabei gelernt habe

**Erst messen, dann entscheiden.** Hergeleitete Werte lagen fast immer daneben,
oft um das Doppelte. Eine Schwelle, die ich aus der menschlichen Schrittfrequenz
abgeleitet hatte, war doppelt so hoch wie das mechanisch Moegliche — mit dem
Ergebnis, dass ueber 16.000 Durchlaeufe kein einziger Schritt gezaehlt wurde.

**Bei Vergleichen genau eine Sache aendern.** Sonst ist ein Unterschied
niemandem zuzuordnen. Diese Regel habe ich mehrfach verletzt und jedes Mal
dafuer bezahlt.

**Ein einzelner Messpunkt zeigt keine Richtung.** Es ist normal, dass ein Stand
erst schlechter wird, bevor er besser wird.

**Dranbleiben statt nachstellen.** Die Laeufe, die am weitesten kamen, waren
die, an denen ich zwischendurch nichts veraendert habe.

**Beobachten schlaegt Rechnen.** Die groessten Fortschritte kamen nicht aus den
Zahlen, sondern daraus, dem Roboter in der 3D-Ansicht zuzusehen und zu
erkennen, woran es hakt.
