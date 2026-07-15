# DAX Measure-System: Aktivitätstypen in fact_timepositions

Grundstruktur für ein Measure-System, das flexibel auf Berichtsfilter reagiert,
wo es soll, und stur bei festen Werten bleibt, wo es soll — bei voller
Transparenz im Bericht.

## 1. Datenmodell-Empfehlung

Aktivitätstyp sollte als eigene Dimensionstabelle `dim_Aktivitätstyp` modelliert
werden (1:n-Beziehung zu `fact_timepositions`), statt nur als Textspalte in der
Faktentabelle zu leben. Grund: Nur so lässt sich die "nicht summierbar"-Regel
zentral pflegen, statt sie in jedem Measure zu wiederholen.

| Spalte | Zweck |
|---|---|
| `Aktivitätstyp` | Schlüssel / Anzeigetext, z. B. "Produktiv", "Urlaub", "Krankheit" |
| `SummierGruppe` | Gruppen, deren Mitglieder addiert werden dürfen (z. B. "Anwesenheit" vs. "Abwesenheit") |
| `Sortierung` | Anzeigereihenfolge in Slicern/Visuals |

Falls ihr (noch) keine eigene Dimension habt: Alle Muster unten funktionieren
sinngemäß auch direkt auf `fact_timepositions[Aktivitätstyp]` — dann
`REMOVEFILTERS( dim_Aktivitätstyp )` durch
`REMOVEFILTERS( fact_timepositions[Aktivitätstyp] )` ersetzen. Die Dimension
lohnt sich aber, sobald ihr Gruppen/Sortierung/Übersetzungen pflegen wollt.

## 2. Ebene 0 — Basis-Measures (technisch, Display Folder "00 Basis")

```dax
Menge Basis =
SUM ( fact_timepositions[Menge] )

Anzahl Buchungen =
COUNTROWS ( fact_timepositions )
```

Alle anderen Measures bauen auf `[Menge Basis]` auf — nie direkt `SUM(...)`
wiederholen, sonst driftet die Logik bei späteren Anpassungen auseinander.

## 3. Konsistenz-Guard — verhindert unerlaubtes Summieren

```dax
Anzahl SummierGruppen im Kontext =
COUNTROWS (
    DISTINCT ( dim_Aktivitätstyp[SummierGruppe] )
)

Aktivitätstyp-Auswahl gültig =
[Anzahl SummierGruppen im Kontext] <= 1
```

Sobald der Berichtsfilter/Slicer Aktivitätstypen aus mehr als einer
`SummierGruppe` gleichzeitig aktiv hat, ist die Auswahl "ungültig" — das
flexible Measure aus Abschnitt 4 nutzt das, um keine falsche Summe
auszuweisen.

## 4. Ebene 1 — Flexibles Measure (reagiert auf Berichtsfilter)

```dax
Stunden (flexibel) =
VAR ErgebnisBasis = [Menge Basis]
VAR GueltigeAuswahl = [Aktivitätstyp-Auswahl gültig]
RETURN
    IF (
        NOT GueltigeAuswahl,
        BLANK (),
        ErgebnisBasis
    )

Stunden (flexibel) Hinweis =
IF (
    NOT [Aktivitätstyp-Auswahl gültig],
    "⚠ Ausgewählte Aktivitätstypen dürfen nicht gemeinsam summiert werden.",
    BLANK ()
)
```

`BLANK()` statt eines falschen Wertes verhindert stille Fehlinterpretation;
das Hinweis-Measure macht den Grund im Bericht sichtbar (z. B. als
Kartentext direkt neben der KPI, oder als Tooltip).

Dieses Measure reagiert ganz normal auf jeden Filter/Slicer auf
`dim_Aktivitätstyp` — kein zusätzlicher Code nötig, das ist Standard-DAX-
Filterkontext-Verhalten.

## 5. Ebene 2 — Fixe Vergleichs-Measures (ignorieren den Aktivitätstyp-Filter)

Diese Measures sollen als stabile Referenzwerte dienen, unabhängig davon,
was der Anwender im Bericht bei Aktivitätstyp ausgewählt hat. Muster:
`REMOVEFILTERS` auf die Dimension, dann die gewünschte feste Menge explizit
per `IN` reinschreiben. Alle anderen Filter (Datum, Mitarbeiter, Projekt …)
bleiben unangetastet, weil nur die Aktivitätstyp-Dimension zurückgesetzt wird.

```dax
Stunden Produktiv (fix) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( dim_Aktivitätstyp ),
    dim_Aktivitätstyp[Aktivitätstyp] IN { "Produktiv", "Projektarbeit" }
)

Stunden Abwesenheit (fix) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( dim_Aktivitätstyp ),
    dim_Aktivitätstyp[Aktivitätstyp] IN { "Urlaub", "Krankheit", "Feiertag" }
)

Stunden Gesamt (fix, alle Typen) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( dim_Aktivitätstyp )
)
```

Jede neue Kombination ist eine weitere Kopie dieses Musters mit eigener
`IN {...}`-Liste — bewusst kein Parameter/Switch, damit jede Kombination als
eigenständiges, benanntes, testbares Measure existiert.

## 6. Ebene 3 — Darauf aufbauende KPIs

```dax
Anteil Produktiv an Gesamt (fix) =
DIVIDE (
    [Stunden Produktiv (fix)],
    [Stunden Gesamt (fix, alle Typen)]
)

Abweichung Flexibel vs. Produktiv (fix) =
[Stunden (flexibel)] - [Stunden Produktiv (fix)]
```

So entstehen Vergleichs-KPIs, bei denen die linke Seite auf Nutzerauswahl
reagiert und die rechte Seite als stabiler Anker fungiert — genau der
Vergleichsfall aus der Anforderung.

## 7. Transparenz im Bericht

Zwei Textmeasures, die als dynamischer Titel/Untertitel in Visuals
eingebunden werden, damit jederzeit sichtbar ist, worauf sich eine Zahl
gerade bezieht:

```dax
Ausgewählte Aktivitätstypen (Text) =
VAR AnzahlAusgewaehlt = COUNTROWS ( VALUES ( dim_Aktivitätstyp[Aktivitätstyp] ) )
VAR AnzahlGesamt = COUNTROWS ( ALL ( dim_Aktivitätstyp[Aktivitätstyp] ) )
RETURN
    IF (
        AnzahlAusgewaehlt = AnzahlGesamt,
        "alle Aktivitätstypen",
        CONCATENATEX (
            VALUES ( dim_Aktivitätstyp[Aktivitätstyp] ),
            dim_Aktivitätstyp[Aktivitätstyp],
            ", "
        )
    )

Stunden (flexibel) – Titel =
"Stunden (" & [Ausgewählte Aktivitätstypen (Text)] & ")"

Hinweis Fixmessung =
"Fixe Vergleichsbasis — unabhängig von der Aktivitätstyp-Auswahl im Bericht."
```

`[Stunden (flexibel) – Titel]` als dynamischen Titel des Visuals verwenden;
`[Hinweis Fixmessung]` als Tooltip-Zusatzfeld oder Textbox neben jedem fixen
Measure. Damit ist auf einen Blick klar, welches Measure filterreagierend
ist und welches nicht — ohne dass der Anwender den DAX-Code lesen muss.

## 8. Namens- und Ordnungskonvention (Skalierung auf mehr Measures)

| Suffix/Ordner | Bedeutung |
|---|---|
| `... Basis` | Display Folder "00 Basis" — nie direkt im Bericht verwenden |
| `... (flexibel)` | Display Folder "01 Flexibel" — reagiert auf Aktivitätstyp-Filter |
| `... (fix)` / `... (fix, ...)` | Display Folder "02 Fix" — ignoriert Aktivitätstyp-Filter, reagiert auf alle anderen Filter |
| `... – Titel` / `Hinweis ...` | Display Folder "03 Transparenz" — Textmeasures für Titel/Tooltip |

Diese Konvention konsequent durchhalten, dann bleibt das System auch bei
20+ Measures und mehreren Aktivitätstyp-Kombinationen nachvollziehbar.
