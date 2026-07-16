# DAX Measure-System: Aktivitätstypen ohne separate Dimensionstabelle

Variante von [`aktivitaetstyp-measures.md`](./aktivitaetstyp-measures.md), die
bewusst auf eine zusätzliche `dim_Aktivitätstyp`-Tabelle verzichtet. Alle
Measures arbeiten direkt auf der Spalte `fact_timepositions[Aktivitätstyp]`.
Fachlich identisches Ergebnis, andere Wartungs-Tradeoffs — siehe Abschnitt 6.

## 1. Ausgangsdaten

`fact_timepositions` enthält:

- `Menge` — gebuchte Stundenzahl
- `Aktivitätstyp` — Text, einer von: `KommenGehen`, `Abwesenheit`,
  `Projektstunden ADLON Intern ohne KommenGehen`,
  `Projektstunden Kunde ohne TicketZeiterfassung`,
  `TicketZeiterfassung Kunde`, `TicketZeiterfassung ADLON`, `Zeitausgleich`
- `IstAktuelleZeile` — "Ja"/"Nein", trennt aktive von historischen Zeilen

Regeln:
- `IstAktuelleZeile = "Nein"` darf in keinem Measure auftauchen.
- `Zeitausgleich` ist grundsätzlich neutral und darf nie aufsummiert werden
  (auch nicht allein).
- `TicketZeiterfassung Kunde`/`TicketZeiterfassung ADLON` dürfen nie
  gemeinsam mit `KommenGehen` summiert werden.

Es gibt **keine** eigene Dimensionstabelle — jede Gruppierung (z. B. "beide
TicketZeiterfassung-Typen") wird direkt als Werte-Liste in den betroffenen
Measures angegeben.

## 2. Ebene 0 — Basis-Measures (Display Folder "00 Basis")

Beide permanenten Regeln (Zeilenfilter, Zeitausgleich-Ausschluss) werden hier
fest verankert, damit sie nicht in jedem Measure wiederholt werden müssen:

```dax
Menge Basis =
CALCULATE (
    SUM ( fact_timepositions[Menge] ),
    fact_timepositions[IstAktuelleZeile] = "Ja",
    fact_timepositions[Aktivitätstyp] <> "Zeitausgleich"
)

Anzahl Buchungen =
CALCULATE (
    COUNTROWS ( fact_timepositions ),
    fact_timepositions[IstAktuelleZeile] = "Ja",
    fact_timepositions[Aktivitätstyp] <> "Zeitausgleich"
)
```

**Faustregel:** Jedes neue Measure baut auf `[Menge Basis]` /
`[Anzahl Buchungen]` auf, statt erneut direkt `SUM(...)`/`COUNTROWS(...)` auf
`fact_timepositions` aufzurufen — sonst gehen beide Regeln für dieses eine
Measure verloren.

## 3. Konsistenz-Guard — KommenGehen ↔ TicketZeiterfassung

Da es keine `Kategorie`-Spalte gibt, wird die TicketZeiterfassung-Gruppe hier
direkt als Werte-Liste geprüft:

```dax
Enthält "KommenGehen" =
CONTAINS ( VALUES ( fact_timepositions[Aktivitätstyp] ), fact_timepositions[Aktivitätstyp], "KommenGehen" )

Enthält "TicketZeiterfassung" =
CONTAINS ( VALUES ( fact_timepositions[Aktivitätstyp] ), fact_timepositions[Aktivitätstyp], "TicketZeiterfassung Kunde" )
    || CONTAINS ( VALUES ( fact_timepositions[Aktivitätstyp] ), fact_timepositions[Aktivitätstyp], "TicketZeiterfassung ADLON" )

Aktivitätstyp-Auswahl gültig =
NOT (
    [Enthält "KommenGehen"]
        && [Enthält "TicketZeiterfassung"]
)
```

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
    "⚠ KommenGehen und TicketZeiterfassung dürfen nicht gemeinsam summiert werden.",
    BLANK ()
)
```

Unverändert gegenüber der Dimensions-Variante — der Guard liefert dasselbe
Ergebnis, nur seine interne Prüfung läuft direkt gegen die Faktentabelle.

## 5. Ebene 2 — Fixe Vergleichs-Measures (ignorieren den Aktivitätstyp-Filter)

```dax
Stunden KommenGehen (fix) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( fact_timepositions[Aktivitätstyp] ),
    fact_timepositions[Aktivitätstyp] = "KommenGehen"
)

Stunden TicketZeiterfassung (fix) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( fact_timepositions[Aktivitätstyp] ),
    fact_timepositions[Aktivitätstyp] IN { "TicketZeiterfassung Kunde", "TicketZeiterfassung ADLON" }
)

Stunden Projektstunden (fix) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( fact_timepositions[Aktivitätstyp] ),
    fact_timepositions[Aktivitätstyp] IN {
        "Projektstunden ADLON Intern ohne KommenGehen",
        "Projektstunden Kunde ohne TicketZeiterfassung"
    }
)

Stunden Abwesenheit (fix) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( fact_timepositions[Aktivitätstyp] ),
    fact_timepositions[Aktivitätstyp] = "Abwesenheit"
)

Stunden Gesamt (fix, ohne Zeitausgleich) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( fact_timepositions[Aktivitätstyp] )
)
```

Isoliertes Zeitausgleich-Measure — einzige Ausnahme von der Faustregel aus
Abschnitt 2, da es den permanenten Ausschluss bewusst wieder aufhebt:

```dax
Zeitausgleich Stunden (isoliert, nicht summierbar) =
CALCULATE (
    SUM ( fact_timepositions[Menge] ),
    fact_timepositions[IstAktuelleZeile] = "Ja",
    REMOVEFILTERS ( fact_timepositions[Aktivitätstyp] ),
    fact_timepositions[Aktivitätstyp] = "Zeitausgleich"
)
```

## 6. Ebene 3 — Darauf aufbauende KPIs

```dax
Anteil TicketZeiterfassung an Gesamt (fix) =
DIVIDE (
    [Stunden TicketZeiterfassung (fix)],
    [Stunden Gesamt (fix, ohne Zeitausgleich)]
)

Abweichung Flexibel vs. Projektstunden (fix) =
[Stunden (flexibel)] - [Stunden Projektstunden (fix)]
```

## 7. Transparenz im Bericht

```dax
Ausgewählte Aktivitätstypen (Text) =
VAR AnzahlAusgewaehlt = COUNTROWS ( VALUES ( fact_timepositions[Aktivitätstyp] ) )
VAR AnzahlGesamt = COUNTROWS ( ALL ( fact_timepositions[Aktivitätstyp] ) )
RETURN
    IF (
        AnzahlAusgewaehlt = AnzahlGesamt,
        "alle Aktivitätstypen",
        CONCATENATEX (
            VALUES ( fact_timepositions[Aktivitätstyp] ),
            fact_timepositions[Aktivitätstyp],
            ", "
        )
    )

Stunden (flexibel) – Titel =
"Stunden (" & [Ausgewählte Aktivitätstypen (Text)] & ")"

Hinweis Fixmessung =
"Fixe Vergleichsbasis — unabhängig von der Aktivitätstyp-Auswahl im Bericht."
```

## 8. Bewusster Tradeoff dieser Variante

Ohne `dim_Aktivitätstyp` gibt es keine einzige Stelle, an der z. B. "was
zählt als TicketZeiterfassung" zentral gepflegt wird. Diese Werte-Liste
taucht doppelt auf:

- in `Enthält "TicketZeiterfassung"` (Abschnitt 3)
- in `Stunden TicketZeiterfassung (fix)` (Abschnitt 5)

Gleiches gilt für die Projektstunden-Gruppierung. **Kommt ein neuer
Aktivitätstyp dazu, der zu einer dieser Gruppen gehört** (z. B. eine dritte
TicketZeiterfassung-Variante), müssen alle betroffenen Stellen manuell
gefunden und ergänzt werden — es gibt keine Kompilier- oder Auswertungszeit-
Prüfung, die eine vergessene Stelle anzeigt.

Das ist bei aktuell 7 Aktivitätstypen und 2 Regeln gut überschaubar. Sobald
mehr Kombinationen oder mehr Regeln dazukommen, empfiehlt sich der Wechsel
zur Variante mit `dim_Aktivitätstyp` (siehe
[`aktivitaetstyp-measures.md`](./aktivitaetstyp-measures.md)) — dort steht
die Gruppierung einmal in der `Kategorie`-Spalte, und jedes Measure verweist
nur noch darauf.

## 9. Namens- und Ordnungskonvention

Identisch zur Dimensions-Variante:

| Suffix/Ordner | Bedeutung |
|---|---|
| `... Basis` | Display Folder "00 Basis" — nie direkt im Bericht verwenden |
| `... (flexibel)` | Display Folder "01 Flexibel" — reagiert auf Aktivitätstyp-Filter |
| `... (fix)` / `... (fix, ...)` | Display Folder "02 Fix" — ignoriert Aktivitätstyp-Filter |
| `... (isoliert, nicht summierbar)` | Display Folder "02 Fix" — steht nie neben anderen Summen |
| `... – Titel` / `Hinweis ...` | Display Folder "03 Transparenz" — Textmeasures für Titel/Tooltip |
