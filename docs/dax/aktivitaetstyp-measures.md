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
| `Aktivitätstyp` | Schlüssel / Anzeigetext (siehe konkrete Werte unten) |
| `Kategorie` | Fachliche Einordnung — Basis für die gezielten Verbots-Prüfungen in Abschnitt 3 |
| `Sortierung` | Anzeigereihenfolge in Slicern/Visuals |

Konkrete Belegung für die vorliegenden Aktivitätstypen:

| Aktivitätstyp | Kategorie |
|---|---|
| KommenGehen | KommenGehen |
| Abwesenheit | Abwesenheit |
| Projektstunden ADLON Intern ohne KommenGehen | Projektstunden |
| Projektstunden Kunde ohne TicketZeiterfassung | Projektstunden |
| TicketZeiterfassung Kunde | TicketZeiterfassung |
| TicketZeiterfassung ADLON | TicketZeiterfassung |
| Zeitausgleich | Zeitausgleich |

Wichtig: `Kategorie` ist **keine** strikte Partition im Sinne von "nur gleiche
Kategorie ist summierbar, alles andere verboten". Sie ist nur die fachliche
Einordnung, auf der die punktuellen Regeln aus Abschnitt 3 aufsetzen. Die
Namensgebung der Projektstunden-Typen ("... ohne KommenGehen" / "... ohne
TicketZeiterfassung") deutet stark darauf hin, dass diese bewusst so
geschnitten wurden, dass sie **ohne Doppelzählung** neben KommenGehen bzw.
TicketZeiterfassung stehen dürfen — es handelt sich also um gezielte
Inkompatibilitäten, nicht um eine generelle Gruppentrennung.

Falls ihr (noch) keine eigene Dimension habt: Alle Muster unten funktionieren
sinngemäß auch direkt auf `fact_timepositions[Aktivitätstyp]` — dann
`REMOVEFILTERS( dim_Aktivitätstyp )` durch
`REMOVEFILTERS( fact_timepositions[Aktivitätstyp] )` ersetzen. Die Dimension
lohnt sich aber, sobald ihr Kategorien/Sortierung/Übersetzungen pflegen wollt.

Zusätzlich gibt es in `fact_timepositions` die Spalte `IstAktuelleZeile`
("Ja"/"Nein"), die aktive von historischen Buchungszeilen trennt. Diese Regel
gilt **ausnahmslos für jedes Measure** im System — sie wird deshalb nicht in
jedem einzelnen Measure wiederholt, sondern einmalig an der Wurzel
(Basis-Measures) verankert. Siehe Abschnitt 2.

## 2. Ebene 0 — Basis-Measures (technisch, Display Folder "00 Basis")

Zwei Dinge gehören fest in die Basis-Measures, nicht in einzelne
Aufbau-Measures:

1. **`IstAktuelleZeile = "Ja"`** — muss ausnahmslos überall gelten.
2. **`Kategorie <> "Zeitausgleich"`** — Zeitausgleich ist laut Vorgabe
   "grundsätzlich neutral" und darf **niemals** aufsummiert werden, auch
   nicht allein. Das ist keine kontextabhängige Prüfung wie bei
   KommenGehen/TicketZeiterfassung (Abschnitt 3), sondern ein permanenter
   Ausschluss — deshalb gehört er strukturell an dieselbe Stelle wie der
   Zeilenfilter.

Ein CALCULATE-Filterargument auf einer Spalte ersetzt jeden vorher
bestehenden Filter auf exakt dieser Spalte (statt ihn nur einzuschränken) —
d. h. selbst wenn irgendwo im Bericht nach `IstAktuelleZeile = "Nein"` oder
nach `Kategorie = "Zeitausgleich"` gefiltert würde, gewinnt hier trotzdem die
Basis-Regel. Der Filter ist damit nicht nur Standard, sondern auch gegen
Fehlbedienung robust.

```dax
Menge Basis =
CALCULATE (
    SUM ( fact_timepositions[Menge] ),
    fact_timepositions[IstAktuelleZeile] = "Ja",
    dim_Aktivitätstyp[Kategorie] <> "Zeitausgleich"
)

Anzahl Buchungen =
CALCULATE (
    COUNTROWS ( fact_timepositions ),
    fact_timepositions[IstAktuelleZeile] = "Ja",
    dim_Aktivitätstyp[Kategorie] <> "Zeitausgleich"
)
```

Alle anderen Measures bauen auf `[Menge Basis]` bzw. `[Anzahl Buchungen]`
auf — nie direkt `SUM(...)` oder `COUNTROWS(...)` auf `fact_timepositions`
wiederholen. Dadurch erben alle nachgelagerten Measures (flexibel, fix, KPI)
automatisch beide Regeln, ohne sie erneut angeben zu müssen — auch die
fixen Measures in Abschnitt 4, obwohl sie `REMOVEFILTERS( dim_Aktivitätstyp )`
verwenden: Das setzt nur *user-gesetzte* Filter auf der Aktivitätstyp-
Dimension zurück; der in `[Menge Basis]` fest verankerte Zeitausgleich-
Ausschluss und der Zeilenfilter bleiben davon unberührt, weil sie Teil des
Measures selbst sind, nicht Teil des externen Filterkontexts.

**Faustregel für jedes neue Measure:** Wenn es Zeilen aus `fact_timepositions`
aggregiert, muss es entweder auf `[Menge Basis]` / `[Anzahl Buchungen]`
aufbauen, oder — falls das aus triftigem Grund nicht geht — beide Filter
(`IstAktuelleZeile = "Ja"`, `Kategorie <> "Zeitausgleich"`) explizit selbst
mitführen. Einzige zulässige Ausnahme: das isolierte Zeitausgleich-Measure
in Abschnitt 4, das den Ausschluss bewusst und sichtbar wieder aufhebt.

## 3. Konsistenz-Guard — gezielte Verbots-Prüfung (nicht Gruppenzählung)

Die einzige bekannte punktuelle Regel: TicketZeiterfassung (Kunde oder
ADLON) darf nie gemeinsam mit KommenGehen ausgewertet werden. Das wird
**nicht** über "Anzahl unterschiedlicher Kategorien im Kontext" geprüft
(das würde z. B. `KommenGehen` + `Abwesenheit` fälschlich blockieren, obwohl
dafür keine Regel existiert) — sondern gezielt: Sind *beide* konkreten
Kategorien gleichzeitig im Filterkontext aktiv?

```dax
Enthält Kategorie "KommenGehen" =
CONTAINS ( VALUES ( dim_Aktivitätstyp[Kategorie] ), dim_Aktivitätstyp[Kategorie], "KommenGehen" )

Enthält Kategorie "TicketZeiterfassung" =
CONTAINS ( VALUES ( dim_Aktivitätstyp[Kategorie] ), dim_Aktivitätstyp[Kategorie], "TicketZeiterfassung" )

Aktivitätstyp-Auswahl gültig =
NOT (
    [Enthält Kategorie "KommenGehen"]
        && [Enthält Kategorie "TicketZeiterfassung"]
)
```

Kommt später eine weitere punktuelle Regel dazu, wird eine weitere
`Enthält Kategorie "..."`-Prüfung ergänzt und per `&&` in
`Aktivitätstyp-Auswahl gültig` eingebunden — die bestehenden Regeln bleiben
unangetastet. Bei vielen Regeln lohnt sich eine kleine Regel-Tabelle
(`KategorieA` / `KategorieB`) mit einer Iterator-Prüfung; bei zwei, drei
Regeln ist die explizite Variante klarer lesbar und leichter zu debuggen.

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

`BLANK()` statt eines falschen Wertes verhindert stille Fehlinterpretation;
das Hinweis-Measure macht den Grund im Bericht sichtbar (z. B. als
Kartentext direkt neben der KPI, oder als Tooltip).

Weil `[Menge Basis]` bereits `Kategorie <> "Zeitausgleich"` fest eingebaut
hat, liefert `[Stunden (flexibel)]` auch dann `0`/`BLANK()` für Zeitausgleich,
wenn jemand versucht, ausschließlich "Zeitausgleich" im Slicer auszuwählen —
konsistent mit der Vorgabe, dass Zeitausgleich grundsätzlich neutral ist und
nie in einer Summe auftaucht. Um Zeitausgleich trotzdem sichtbar zu machen,
gibt es das dedizierte Measure am Ende von Abschnitt 5.

## 5. Ebene 2 — Fixe Vergleichs-Measures (ignorieren den Aktivitätstyp-Filter)

Diese Measures dienen als stabile Referenzwerte, unabhängig davon, was der
Anwender im Bericht bei Aktivitätstyp ausgewählt hat. Muster: `REMOVEFILTERS`
auf die Dimension, dann die gewünschte feste Kategorie/Menge explizit
reinschreiben. Alle anderen Filter (Datum, Mitarbeiter, Projekt …) bleiben
unangetastet, weil nur die Aktivitätstyp-Dimension zurückgesetzt wird.

```dax
Stunden KommenGehen (fix) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( dim_Aktivitätstyp ),
    dim_Aktivitätstyp[Kategorie] = "KommenGehen"
)

Stunden TicketZeiterfassung (fix) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( dim_Aktivitätstyp ),
    dim_Aktivitätstyp[Kategorie] = "TicketZeiterfassung"
)

Stunden Projektstunden (fix) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( dim_Aktivitätstyp ),
    dim_Aktivitätstyp[Kategorie] = "Projektstunden"
)

Stunden Abwesenheit (fix) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( dim_Aktivitätstyp ),
    dim_Aktivitätstyp[Kategorie] = "Abwesenheit"
)

Stunden Gesamt (fix, ohne Zeitausgleich) =
CALCULATE (
    [Menge Basis],
    REMOVEFILTERS ( dim_Aktivitätstyp )
)
```

`Stunden Gesamt (fix, ohne Zeitausgleich)` braucht keinen expliziten
Zeitausgleich-Ausschluss mehr, weil das schon in `[Menge Basis]` fest
verankert ist (Abschnitt 2) — der Name macht diesen Fakt trotzdem sichtbar,
damit niemand im Bericht denkt, hier wäre wirklich "alles" enthalten.

Jede neue Kombination ist eine weitere Kopie dieses Musters — bewusst kein
Parameter/Switch, damit jede Kombination als eigenständiges, benanntes,
testbares Measure existiert.

**Isoliertes Zeitausgleich-Measure** — die einzige zulässige Ausnahme von der
Faustregel aus Abschnitt 2, weil es den permanenten Ausschluss bewusst und
sichtbar wieder aufhebt:

```dax
Zeitausgleich Stunden (isoliert, nicht summierbar) =
CALCULATE (
    SUM ( fact_timepositions[Menge] ),
    fact_timepositions[IstAktuelleZeile] = "Ja",
    REMOVEFILTERS ( dim_Aktivitätstyp ),
    dim_Aktivitätstyp[Kategorie] = "Zeitausgleich"
)
```

Der Name signalisiert im Bericht unmissverständlich: Dieser Wert darf mit
keinem anderen Stunden-Measure addiert werden — er steht bewusst allein.

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
Measure; `[Stunden (flexibel) Hinweis]` (Abschnitt 4) als sichtbaren
Warnhinweis, sobald KommenGehen und TicketZeiterfassung gleichzeitig
ausgewählt sind. Damit ist auf einen Blick klar, welches Measure
filterreagierend ist, welches nicht, und wann eine Auswahl fachlich
unzulässig ist — ohne dass der Anwender den DAX-Code lesen muss.

## 8. Namens- und Ordnungskonvention (Skalierung auf mehr Measures)

| Suffix/Ordner | Bedeutung |
|---|---|
| `... Basis` | Display Folder "00 Basis" — nie direkt im Bericht verwenden |
| `... (flexibel)` | Display Folder "01 Flexibel" — reagiert auf Aktivitätstyp-Filter |
| `... (fix)` / `... (fix, ...)` | Display Folder "02 Fix" — ignoriert Aktivitätstyp-Filter, reagiert auf alle anderen Filter |
| `... (isoliert, nicht summierbar)` | Display Folder "02 Fix" — bewusste Ausnahme, steht nie neben anderen Summen |
| `... – Titel` / `Hinweis ...` | Display Folder "03 Transparenz" — Textmeasures für Titel/Tooltip |

Diese Konvention konsequent durchhalten, dann bleibt das System auch bei
20+ Measures und mehreren Aktivitätstyp-Kombinationen nachvollziehbar.

**Wichtigste Regeln dabei:**
- Jedes neue Measure muss auf `[Menge Basis]` / `[Anzahl Buchungen]`
  aufbauen (Abschnitt 2) statt erneut direkt auf `fact_timepositions` zu
  aggregieren — sonst gehen der `IstAktuelleZeile = "Ja"`-Filter und der
  Zeitausgleich-Ausschluss für dieses eine Measure verloren, ohne dass das
  im Bericht auffällt.
- Neue punktuelle Verbots-Regeln (wie KommenGehen ↔ TicketZeiterfassung)
  werden als zusätzliche `Enthält Kategorie "..."`-Prüfung in
  `Aktivitätstyp-Auswahl gültig` ergänzt (Abschnitt 3) — nicht als weitere
  Gruppen-Spalte, da das bei punktuellen statt vollständig disjunkten
  Regeln zu falschen Blockaden führt.
