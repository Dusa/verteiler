# Verteiler

Dieses Repository hält Dateien, die in anderen Repositories gleich sein sollen, und den wiederverwendbaren Workflow `Auto-Changelog Generator`. Andere Repositories rufen den Workflow auf. Er kopiert die gemeinsamen Dateien ins Ziel-Repository, entfernt dort Dateien, die im Verteiler weggefallen sind, und schreibt dort Changelog sowie Versionsstand.

Der Workflow liegt unter `.github/workflows/changelog_gen.yml`. Er wird nicht mitkopiert. Ziel-Repositories binden ihn ein, sie enthalten ihn nicht selbst.

Im Repository `Dusa/verteiler` läuft der Job nicht. `github.repository` ist dort `Dusa/verteiler`, bei `push` und bei `workflow_dispatch`. Bei `workflow_call` gehört `github.repository` zum aufrufenden Repository, andere Repositories laufen also normal. Dadurch entsteht hier kein Changelog, das der Kopierschritt in die Ziele schreiben könnte.

## Ablauf

Der Job `update-changelog` läuft auf `ubuntu-latest`, sofern `github.repository` nicht `Dusa/verteiler` ist. Für alle Schritte gelten `LC_ALL=C.UTF-8`, `LANG=C.UTF-8` und die Zeitzone `Europe/Vienna`.

1. `actions/checkout@v4` holt das Repository, in dem der Lauf ausgelöst wurde, mit voller Historie (`fetch-depth: 0`). Bei `workflow_call` ist das das aufrufende Repository, weil der `github`-Kontext zum Aufrufer gehört. Bei `push` und `workflow_dispatch` wäre es dieses Repository; der Job ist hier ausgenommen.
2. Ein zweites Checkout holt `Dusa/verteiler` vom Branch `main` nach `_verteiler_temp`. Die gemeinsamen Dateien kommen damit immer von `main`, auch wenn der Workflow selbst über einen anderen Ref eingebunden ist.
3. Die zentralen Dateien werden ins Ziel-Repository kopiert. Dieselbe Namensliste speist `find` und `rsync`. `_verteiler_temp` wird danach gelöscht. `rsync -avm` läuft ohne `--delete`: was nur im Ziel liegt, bleibt. `-m` legt keine leeren Ordner an, die nur ausgeschlossene Dateien enthalten hätten.
4. Liegt schon eine `.verteiler-manifest` im Ziel, wird jeder Pfad daraus entfernt, der in der neuen Liste fehlt. Ohne Manifest wird nichts gelöscht, die Liste wird nur angelegt. Details stehen unter „Entfernen im Ziel“.
5. Die Betreffzeilen der Commits werden gesammelt, zusammen mit Datum `TT.MM.JJJJ` (`date +"%d.%m.%Y"`) und Build `JJJJMMTT_HHMM` (`date +"%Y%m%d_%H%M"`). `EVENT_BEFORE` ist `${{ github.event.before }}`, `EVENT_SHA` ist `${{ github.sha }}`; beide kommen über `env` in die Shell, nicht als direkte Interpolation.
   - Ist `EVENT_BEFORE` gesetzt und nicht die Null-SHA `0000000000000000000000000000000000000000`, nimmt der Lauf `git log "$EVENT_BEFORE".."$EVENT_SHA" --no-merges --pretty=%s`. Schlägt das fehl, nimmt er die letzten drei Nicht-Merge-Commits, ebenfalls nur die Betreffzeile.
   - Ist `EVENT_BEFORE` leer oder die Null-SHA, nimmt der Lauf `git log -3 --no-merges --pretty=%s`. Leer ist `before` bei `workflow_dispatch`. Die Null-SHA setzt GitHub beim ersten Push eines neuen Branches.
   - Betreffzeilen, die mit `chore: auto-update changelog` beginnen, entfallen. Das ist der Commit des Bots. Leere Zeilen entfallen ebenfalls. Die übrigen Zeilen stehen in `msg.txt`, eine Betreffzeile pro Zeile. Der Schritt gibt nur `date` und `build` aus.
6. Jede verbleibende Betreffzeile wird ein eigener Changelog-Eintrag. `CHANGELOG.md` und `CHANGELOG.html` werden geschrieben. Liegt `global_settings.php` oder `global_config.php` im Ziel, werden die Versionskonstanten gesetzt. Die beiden PHP-Dateien kommen nicht aus dem Verteiler.
7. Zum Index kommen `CHANGELOG.md`, `CHANGELOG.html`, `global_settings.php` und `global_config.php`, sofern die jeweilige PHP-Datei existiert, jede Datei aus `.verteiler-manifest` und die Manifest-Datei selbst. Löschungen aus dem Kopierschritt sind bereits mit `git rm` gestaged. Gibt es gestagte Änderungen, committet sie `NudlDudl Bot <nudldudl.bot@users.noreply.github.com>` mit der Nachricht `chore: auto-update changelog and sync files [skip ci]` und pusht. `[skip ci]` verhindert, dass dieser Push den Workflow erneut startet. Ohne Änderungen endet der Schritt ohne Commit.

## Kopierte Dateien

Kopiert wird jede Datei aus `Dusa/verteiler` außer diesen Namen. Der Name gilt in jedem Verzeichnis, nicht nur in der Wurzel. `find` und `rsync` bekommen dieselbe Liste:

- `.git`
- `.github`
- `README.md`
- `CHANGELOG.md`
- `CHANGELOG.html`
- `msg.txt`
- `temp.md`
- `entry_line.txt`
- `verteiler_files.txt`
- `global_settings.php`
- `global_config.php`
- `.verteiler-manifest`

`msg.txt`, `temp.md`, `entry_line.txt` und `verteiler_files.txt` sind Zwischendateien des Laufs. `verteiler_files.txt` wird nicht mehr geschrieben; die Stelle hat `.verteiler-manifest` übernommen. Die Zwischendateien stehen trotzdem auf der Liste, damit ein versehentlicher Commit im Verteiler sie nicht verteilt.

`global_settings.php` und `global_config.php` werden nicht verteilt. Sie gehören zum Ziel-Repository. Der Workflow ändert dort nur `APP_VERSION_DATE` und `APP_VERSION_BUILD`. Ein Exemplar aus dem Verteiler würde die übrige Konfiguration des Ziels überschreiben. In diesem Repository liegen die beiden Dateien nicht. Dieselbe Entscheidung steht auf der Schutzliste beim Löschen: ein altes Manifest kann sie nicht aus dem Ziel entfernen.

`.verteiler-manifest` entsteht im Ziel. Ein Exemplar aus dem Verteiler würde die Merkliste des Ziels ersetzen und beim nächsten Lauf die falschen Dateien löschen.

Die Liste im Workflow:

```bash
exclude_names=(
  .git
  .github
  README.md
  CHANGELOG.md
  CHANGELOG.html
  msg.txt
  temp.md
  entry_line.txt
  verteiler_files.txt
  global_settings.php
  global_config.php
  .verteiler-manifest
)
for name in "${exclude_names[@]}"; do
  find_args+=( -not -path "*/${name}/*" -not -name "$name" )
  rsync_args+=( --exclude="$name" )
done
```

`find` schließt damit den Namen selbst und alles darunter aus. `rsync` mit `--exclude` ohne führenden Schrägstrich macht dasselbe für Datei und Verzeichnis. Vor dem Kopieren prüft der Schritt, dass kein Treffer einen ausgeschlossenen Namen enthält. Sonst bricht er ab, ohne zu kopieren oder zu löschen.

Diese README liegt deshalb nur hier und überschreibt keine README im Ziel. `CHANGELOG.md` und `CHANGELOG.html` des Verteilers würden sonst den Changelog des Ziels ersetzen; der nächste Eintrag hinge dann nur noch an der Historie des Verteilers. Beide Dateien werden hier nicht mehr geführt.

Aktuell werden mitkopiert:

- `.editorconfig`
- `logo_55.bmp`
- `logo_seitlich.bmp`

Neue Dateien im Verteiler werden beim nächsten Lauf mitkopiert, solange ihr Name nicht auf der Ausschlussliste steht.

## Entfernen im Ziel

`.verteiler-manifest` liegt in der Wurzel des Ziels. Jede Zeile ist ein Pfad relativ zur Wurzel, ohne Präfix `./`, sortiert mit `sort -u` in der Locale des Jobs (`C.UTF-8`). Die Datei listet genau die Dateien, die der letzte Lauf aus dem Verteiler übernommen hat. Sich selbst listet sie nicht. Pfade mit Leerzeichen sind eine Zeile; die Schleifen lesen mit `IFS= read -r`. Ein Zeilenumbruch im Pfad bricht den Lauf ab.

Ablauf bei jedem Lauf:

1. Die bisherige Manifest-Datei wird gelesen, bevor `rsync` etwas überschreiben kann. Sie wird nach `sort -u` verglichen. Ein CR am Zeilenende wird vorher entfernt.
2. `find` im Checkout des Verteilers liefert die neue Liste, mit derselben Ausschlussliste wie `rsync`.
3. `rsync -avm` kopiert diese Dateien und lässt leere Ordner weg. Dateien, die nur im Ziel existieren, bleiben.
4. Jeder Pfad, der in der alten Liste steht und in der neuen fehlt, wird mit `git rm -q -f --ignore-unmatch --` entfernt. Danach werden leere Elternverzeichnisse hochgelaufen, bis ein Verzeichnis noch etwas enthält oder die Wurzel erreicht ist. `git rm` nimmt ein leer gewordenes Verzeichnis oft schon mit; was liegen bleibt, entfernt `rmdir`. Beides erscheint in der Job-Ausgabe.
5. Die neue Liste ersetzt `.verteiler-manifest`. Der Commit-Schritt nimmt sie zusammen mit den verteilten Dateien in den Index. `git rm` hat Löschungen schon gestaged.

Gibt es noch keine `.verteiler-manifest`, löscht der Lauf nichts und schreibt nur die neue Liste. Eine vorhandene, leere Datei zählt als Manifest: der letzte Lauf hat nichts verteilt, gelöscht wird nur, was in ihr steht. Der erste Lauf nach dieser Änderung merkt sich den aktuellen Stand. Was vorher aus dem Verteiler kam und dort schon fehlt, bleibt im Ziel, bis es einmal in einem Manifest stand und danach im Verteiler fehlt.

Niemals gelöscht, auch wenn der Pfad im alten Manifest steht:

- `.git/**` und `.github/**`, auch wenn der Ordner tiefer liegt
- `README.md`, `CHANGELOG.md`, `CHANGELOG.html`
- `global_settings.php`, `global_config.php`
- `.verteiler-manifest`

Die fünf Dateinamen gelten in jedem Verzeichnis. Zusätzlich überspringt der Lauf absolute Pfade, leere Zeilen und Pfade mit `..` oder einem Zeilenumbruch. Ein Verzeichnis-Eintrag wird nicht rekursiv gelöscht. Fehlt die Datei schon, meldet der Lauf das und macht weiter. Liegt die Datei nur im Arbeitsbaum und nicht im Index, bleibt sie liegen: `git rm` entfernt sie dann nicht.

Der Job gibt die entfernten Pfade aus, jeweils als Zeile `Entfernt: <Pfad>`. Leere Verzeichnisse meldet er als `Leeres Verzeichnis entfernt: <Pfad>`. Geschützte, unsichere oder schon fehlende Pfade bekommen eine eigene Zeile und werden nicht gelöscht.

## Generierte Dateien

### `CHANGELOG.md`

Pro Tag gibt es eine Überschrift `## TT.MM.JJJJ`. Jeder Nicht-Merge-Commit des Pushes wird ein eigener Eintrag, nur aus der Betreffzeile:

```markdown
* **KATEGORIE** (Build JJJJMMTT_HHMM): Betreff
```

`KATEGORIE` ist das erste Vorkommen von `NEU`, `GEÄNDERT`, `ENTFERNT` oder `BEHOBEN` in dieser Betreffzeile. Fehlt es, wird `GEÄNDERT` verwendet. Ein führendes `[...]` wird aus dieser Zeile entfernt. Der Commit-Rumpf, weitere Zeilen und `Co-authored-by:` kommen nicht in den Changelog. Dieselbe Build-Nummer steht an jedem Eintrag des Laufs. Die Reihenfolge ist die von `git log`: der neueste Commit zuerst.

Gibt es keine Betreffzeile, weil der Bereich leer ist oder nur Bot-Commits enthält, bleibt eine vorhandene `CHANGELOG.md` unverändert. Fehlt sie in dem Fall, legt der Schritt nur die Zeile `# Changelog` an.

Existiert die Tagesüberschrift schon, werden die neuen Einträge direkt darunter eingefügt. Ältere Einträge desselben Tages und aller anderen Tage bleiben stehen. Sonst wird die Datei neu aufgebaut: `# Changelog`, eine Leerzeile, `## TT.MM.JJJJ`, die Einträge, eine Leerzeile, und danach der bisherige Inhalt ab Zeile 2. Fehlt `CHANGELOG.md`, legt der Schritt sie vorher mit der Zeile `# Changelog` an.

Die fertige `CHANGELOG.md` ist UTF-8 mit führendem BOM (`EF BB BF`). `.editorconfig` verlangt `charset = utf-8-bom` für alle Dateien außer YAML. Ein schon vorhandenes BOM wird nicht doppelt geschrieben. Eine unveränderte Datei bekommt dabei kein neues BOM: ohne neue Einträge fasst der Schritt eine vorhandene Datei nicht an.

### `CHANGELOG.html`

Die Datei ist ebenfalls UTF-8 mit führendem BOM und enthält `<meta charset="UTF-8">`. `npx --yes marked` wandelt `CHANGELOG.md` um. Ein BOM am Anfang der Markdown-Datei wird vorher entfernt, damit es nicht in den HTML-Rumpf gerät. Die Seite setzt Farben über `prefers-color-scheme` (hell und dunkel). Jede `h2`-Überschrift wird zu einem `details`-Block mit dem Datum als `id`. Ein Link-Symbol kopiert die Anker-URL. Ohne Hash ist nur der erste Block offen. Mit Hash öffnet sich der passende Block, die anderen schließen sich, und `scrollIntoView({ behavior: "smooth", block: "start" })` springt dorthin.

### Version in PHP

Gibt es `global_settings.php`, wird nur diese Datei angepasst. Sonst `global_config.php`, falls vorhanden. Fehlen beide, überspringt der Schritt die Version. Der Schritt setzt kein BOM vor `<?php`. Ein BOM, das schon am Dateianfang steht, lässt `sed` stehen. Die Datei muss schon im Ziel liegen; der Kopierschritt bringt sie nicht mit.

| Konstante | Wert |
| --- | --- |
| `APP_VERSION_DATE` | `JJJJ-MM-TT` (`date +"%Y-%m-%d"` im Konfigurationsschritt) |
| `APP_VERSION_BUILD` | `JJJJMMTT_HHMM` aus dem Schritt, der die Commit-Betreffe liest |

Eine vorhandene Textstelle `define_if_not_set('APP_VERSION_DATE'` bzw. `define_if_not_set('APP_VERSION_BUILD'` wird mitsamt dem Rest der Zeile ersetzt. Fehlt sie, fügt `sed` sie als neue Zeile 2 ein. Zuerst das Datum, danach der Build. Fehlen beide, schiebt die zweite Einfügung das Datum nach Zeile 3. In einer Datei, die mit `<?php` beginnt, sieht das so aus (Werte nur als Formatbeispiel):

```php
<?php
define_if_not_set('APP_VERSION_BUILD', '20261027_1430');
define_if_not_set('APP_VERSION_DATE', '2026-10-27');
```

## Einbindung

Im Ziel-Repository reicht ein Aufrufer. Der `uses`-Ref `@main` wählt die Workflow-Datei. Die kopierten Dateien kommen zusätzlich immer aus dem Checkout von `Dusa/verteiler` mit `ref: main`.

Der aufrufende Job muss `contents: write` setzen, damit der Workflow committen und pushen kann. Dieselbe Berechtigung steht in `changelog_gen.yml`. Ein aufgerufener Workflow kann Rechte des Aufrufers nur beibehalten oder einschränken.

```yaml
name: Changelog

on:
  push:
    branches:
      - main
      - master

jobs:
  changelog:
    permissions:
      contents: write
    uses: Dusa/verteiler/.github/workflows/changelog_gen.yml@main
```

## Auslöser

- `push` auf `master` und `main`, mit `paths-ignore: .github/workflows/**`. Commits, die nur den Workflow ändern, starten ihn nicht. Deshalb lässt sich eine Workflow-Änderung nicht durch einen Push an diese Datei prüfen. In `Dusa/verteiler` startet ein Push anderer Dateien den Workflow, der Job `update-changelog` wird aber übersprungen.
- `workflow_call`, für die Einbindung oben. `github.repository` ist das aufrufende Repository. Aufrufer außer `Dusa/verteiler` führen den Job aus.
- `workflow_dispatch`, ohne Eingaben. `paths-ignore` gilt dafür nicht. In `Dusa/verteiler` ist der Job ausgenommen, ein manueller Lauf schreibt hier also keinen Changelog. `github.event.before` ist bei einem manuellen Lauf im aufrufenden Repository leer; der Schritt verwendet dann die letzten drei Nicht-Merge-Commits, ohne die Bot-Betreffe.

## Commit-Texte

Der Workflow übersetzt Commit-Nachrichten nicht. `git log --no-merges --pretty=%s` schreibt nur die Betreffzeile nach `msg.txt`. Merge-Commits entfallen. Commits, deren Betreff mit `chore: auto-update changelog` beginnt, entfallen. Jede verbleibende Zeile wird ein Changelog-Eintrag. Beginnt sie mit einer eckigen Klammer, wird nur dieses führende Kennzeichen entfernt. Der Rumpf der Commit-Nachricht wird nicht angehängt.
