# Verteiler

Dieses Repository hält Dateien, die in anderen Repositories gleich sein sollen, und den wiederverwendbaren Workflow `Auto-Changelog Generator`. Andere Repositories rufen den Workflow auf. Er kopiert die gemeinsamen Dateien ins Ziel-Repository und schreibt dort Changelog sowie Versionsstand.

Der Workflow liegt unter `.github/workflows/changelog_gen.yml`. Er wird nicht mitkopiert. Ziel-Repositories binden ihn ein, sie enthalten ihn nicht selbst.

## Ablauf

Der Job `update-changelog` läuft auf `ubuntu-latest`. Für alle Schritte gelten `LC_ALL=C.UTF-8`, `LANG=C.UTF-8` und die Zeitzone `Europe/Vienna`.

1. `actions/checkout@v4` holt das Repository, in dem der Lauf ausgelöst wurde, mit voller Historie (`fetch-depth: 0`). Bei `workflow_call` ist das das aufrufende Repository, weil der `github`-Kontext zum Aufrufer gehört. Bei `push` und `workflow_dispatch` ist es dieses Repository.
2. Ein zweites Checkout holt `Dusa/verteiler` vom Branch `main` nach `_verteiler_temp`. Die gemeinsamen Dateien kommen damit immer von `main`, auch wenn der Workflow selbst über einen anderen Ref eingebunden ist.
3. Die zentralen Dateien werden ins Ziel-Repository kopiert. `_verteiler_temp` wird danach gelöscht. Die Dateiliste bleibt vorerst in `verteiler_files.txt`.
4. Die Commit-Nachrichten werden gesammelt, zusammen mit Datum `TT.MM.JJJJ` (`date +"%d.%m.%Y"`) und Build `JJJJMMTT_HHMM` (`date +"%Y%m%d_%H%M"`). `EVENT_BEFORE` ist `${{ github.event.before }}`, `EVENT_SHA` ist `${{ github.sha }}`; beide kommen über `env` in die Shell, nicht als direkte Interpolation.
   - Ist `EVENT_BEFORE` gesetzt und nicht die Null-SHA `0000000000000000000000000000000000000000`, nimmt der Lauf `git log "$EVENT_BEFORE".."$EVENT_SHA" --no-merges --pretty=%B`. Schlägt das fehl, nimmt er die letzten drei Nicht-Merge-Commits.
   - Ist `EVENT_BEFORE` leer oder die Null-SHA, nimmt der Lauf ebenfalls `git log -3 --no-merges --pretty=%B`. Leer ist `before` bei `workflow_dispatch`. Die Null-SHA setzt GitHub beim ersten Push eines neuen Branches.
5. Die Nachrichten werden übersetzt, siehe unten.
6. `CHANGELOG.md` und `CHANGELOG.html` werden geschrieben. Liegt `global_settings.php` oder `global_config.php` im Ziel, werden die Versionskonstanten gesetzt.
7. Zum Index kommen `CHANGELOG.md`, `CHANGELOG.html`, `global_settings.php` und `global_config.php`, sofern die jeweilige PHP-Datei existiert, sowie jede Datei aus `verteiler_files.txt`. Die Liste wird danach gelöscht. Gibt es gestagte Änderungen, committet sie `NudlDudl Bot <nudldudl.bot@users.noreply.github.com>` mit der Nachricht `chore: auto-update changelog and sync files [skip ci]` und pusht. `[skip ci]` verhindert, dass dieser Push den Workflow erneut startet. Ohne Änderungen endet der Schritt ohne Commit.

## Kopierte Dateien

Kopiert wird jede Datei aus `Dusa/verteiler` außer:

- `.git/`
- `.github/`
- `README.md` (auch in Unterordnern; der Dateiname ist ausgeschlossen)

Die Liste entsteht so:

```bash
find . -type f -not -path "*/\.git/*" -not -path "*/\.github/*" -not -name "README.md"
rsync -av --exclude='.git' --exclude='.github' --exclude='README.md' _verteiler_temp/ ./
```

In den doppelten Anführungszeichen bleibt der Backslash vor dem Punkt stehen. `find` behandelt `\.` als Punkt, schließt also Pfade unter `.git` und `.github` aus.

Diese README liegt deshalb nur hier und überschreibt keine README im Ziel. Aktuell werden mitkopiert:

- `.editorconfig`
- `logo_55.bmp`
- `logo_seitlich.bmp`

Neue Dateien im Verteiler werden beim nächsten Lauf mitkopiert, solange sie nicht unter die drei Ausnahmen fallen.

## Generierte Dateien

### `CHANGELOG.md`

Pro Tag gibt es eine Überschrift `## TT.MM.JJJJ`. Ein Eintrag hat die Form:

```markdown
* **KATEGORIE** (Build JJJJMMTT_HHMM): Text
```

`KATEGORIE` ist `NEU`, `GEÄNDERT`, `ENTFERNT` oder `BEHOBEN`. Fehlt sie im Text, wird `GEÄNDERT` verwendet. Ein führendes `[...]` wird aus dem Text entfernt, der Rest bleibt stehen. Existiert die Tagesüberschrift schon, wird der neue Eintrag direkt darunter eingefügt. Sonst wird die Datei neu aufgebaut: `# Changelog`, eine Leerzeile, `## TT.MM.JJJJ`, der Eintrag, eine Leerzeile, und danach der bisherige Inhalt ab Zeile 2. Fehlt `CHANGELOG.md`, legt der Schritt sie vorher mit der Zeile `# Changelog` an, der Neuaufbau hängt daran nichts weiter an.

### `CHANGELOG.html`

Aus `CHANGELOG.md` erzeugt `npx --yes marked` das HTML. Die Seite setzt Farben über `prefers-color-scheme` (hell und dunkel). Jede `h2`-Überschrift wird zu einem `details`-Block mit dem Datum als `id`. Ein Link-Symbol kopiert die Anker-URL. Ohne Hash ist nur der erste Block offen. Mit Hash öffnet sich der passende Block, die anderen schließen sich, und `scrollIntoView({ behavior: "smooth", block: "start" })` springt dorthin.

### Version in PHP

Gibt es `global_settings.php`, wird nur diese Datei angepasst. Sonst `global_config.php`, falls vorhanden. Fehlen beide, überspringt der Schritt die Version.

| Konstante | Wert |
| --- | --- |
| `APP_VERSION_DATE` | `JJJJ-MM-TT` (`date +"%Y-%m-%d"` im Konfigurationsschritt) |
| `APP_VERSION_BUILD` | `JJJJMMTT_HHMM` aus dem Schritt, der die Commit-Nachrichten liest |

Eine vorhandene Textstelle `define_if_not_set('APP_VERSION_DATE'` bzw. `define_if_not_set('APP_VERSION_BUILD'` wird mitsamt dem Rest der Zeile ersetzt. Fehlt sie, fügt `sed` sie als neue Zeile 2 ein. Zuerst das Datum, danach der Build. Fehlen beide, schiebt die zweite Einfügung das Datum nach Zeile 3. In einer Datei, die mit `<?php` beginnt, sieht das so aus (Werte nur als Formatbeispiel):

```php
<?php
define_if_not_set('APP_VERSION_BUILD', '20261027_1430');
define_if_not_set('APP_VERSION_DATE', '2026-10-27');
```

## Einbindung

Im Ziel-Repository reicht ein Aufrufer. Der `uses`-Ref `@main` wählt die Workflow-Datei. Die kopierten Dateien kommen zusätzlich immer aus dem Checkout von `Dusa/verteiler` mit `ref: main`.

Der aufrufende Job muss `contents: write` und `models: read` setzen. Ein aufgerufener Workflow kann die Rechte des Aufrufers nur beibehalten oder einschränken, nicht erweitern. Dieselben zwei Rechte stehen deshalb auch in `changelog_gen.yml`. Ohne `models: read` am Aufrufer hat der Lauf das Recht nicht, auch wenn die aufgerufene Datei es nennt.

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
      models: read
    uses: Dusa/verteiler/.github/workflows/changelog_gen.yml@main
```

## Auslöser

- `push` auf `master` und `main`, mit `paths-ignore: .github/workflows/**`. Commits, die nur den Workflow ändern, starten ihn nicht. Deshalb lässt sich eine Workflow-Änderung nicht durch einen Push an diese Datei prüfen.
- `workflow_call`, für die Einbindung oben.
- `workflow_dispatch`, ohne Eingaben. Damit lässt sich der Workflow in diesem Repository manuell starten. `paths-ignore` gilt dafür nicht. `github.event.before` ist bei einem manuellen Lauf leer; der Schritt verwendet dann die letzten drei Nicht-Merge-Commits. Ein manueller Lauf schreibt Changelog und Versionsdateien in dieses Repository und pusht sie, wenn sich etwas geändert hat.

## KI-Übersetzung und Fallback

Der Schritt `AI Translation via GitHub Copilot / Models` sendet die Commit-Nachrichten an GitHub Models:

- `POST https://models.inference.ai.azure.com/chat/completions`
- Modell `gpt-4o`
- `Authorization: Bearer` mit `github.token`

Der Systemtext verlangt eine deutsche Fassung der gesamten Nachricht, ignoriert automatische Git-Texte wie `Merge branch` und `Merge pull request`, behält vorhandene Absätze und Listen, setzt in langem Fließtext Absätze, verwendet als Zeilenumbruch nur `<br>` und liefert eine einzige Zeile. Die Zeile muss mit `[NEU]`, `[GEÄNDERT]`, `[ENTFERNT]` oder `[BEHOBEN]` beginnen.

Antwortet die Schnittstelle ohne Text (`choices[0].message.content` leer oder `null`), gilt der Fallback:

1. Zeilen, die ohne Berücksichtigung der Großschreibung mit `Merge` beginnen, werden entfernt.
2. Bleibt nichts übrig, wird `msg.txt` vollständig verwendet.
3. Jede Zeile endet mit `<br>`, das letzte `<br>` entfällt.
4. Davor kommt `[GEÄNDERT] `.

Anschließend werden echte Zeilenumbrüche in beiden Fällen durch Leerzeichen ersetzt, mehrfache Leerzeichen zusammengezogen und der Text getrimmt. `curl` und `jq` beenden den Schritt bei einem Fehler nicht; eine fehlgeschlagene Anfrage landet dadurch im Fallback, ohne dass der Workflow abbricht.

`models: read` ist die Berechtigung, die GitHub für diesen Aufruf mit `github.token` verlangt. Fehlt sie, antwortet die API mit 403, der Schritt sieht keinen Übersetzungstext und nutzt den Fallback. Seit dem 30. Juli 2026 ist die GitHub-Models-Inferenzschnittstelle abgeschaltet. Der Aufruf liefert dann ebenfalls keinen Übersetzungstext, und der Workflow verwendet denselben Fallback. Das Recht bleibt gesetzt, weil der Schritt die Schnittstelle weiterhin so aufruft.
