# Antworten – 04 Den Workflow verstehen

Workflow-Datei: `.github/workflows/microcloud-ci.yml`

## Teil A: Aufbau der Workflow-Datei

**1. Events**
Der Workflow startet bei:
- `push` – bei jedem Push, egal auf welchen Branch
- `pull_request` – bei Pull Requests mit Ziel-Branch `main`
- `workflow_dispatch` – manuell über den Button „Run workflow“ im Actions-Tab

**2. Runner**
Der Job läuft auf `ubuntu-latest`. Das ist eine virtuelle Ubuntu-Maschine, die GitHub bereitstellt (GitHub-hosted Runner). Sie wird für jeden Lauf frisch gestartet und danach gelöscht. Docker und Docker Compose sind schon installiert.

**3. Job**
Der Job heißt `stack-test`, angezeigt als „Compose-Stack testen (Port 8085)“. Er hat 10 Steps und darf höchstens 10 Minuten laufen (`timeout-minutes: 10`). Der Workflow selbst heißt „DoD 2 · Microcloud-CI“, der Zielport steht als Umgebungsvariable `ZIELPORT: "8085"` im Kopf.

**4. `uses:` vs. `run:`**
Nur Step 1 nutzt eine fertige Action: `actions/checkout@v7`. Sie lädt den Code des Repositorys auf den Runner. Alle anderen Steps führen mit `run:` eigene Shell-Befehle aus.

**5. Rechte**
`permissions: contents: read` – der Workflow darf den Code nur lesen, aber nichts ändern, pushen oder veröffentlichen. Das ist das Prinzip der minimalen Rechte (Least Privilege): Wird ein Schritt oder eine Action kompromittiert, kann sie kaum Schaden anrichten.

**6. `if: ${{ !github.event.repository.is_template }}`**
In der Vorlage (Template-Repository) wird der Job übersprungen. Dort gibt es noch keinen Stack, und die Vorlage wäre sonst dauerhaft rot. In meinem Repo ist `is_template` false, deshalb läuft der Job dort immer.

## Teil B: Was wird geprüft?

**7. Welcher Step schlägt zuerst fehl?**
1. Tabulator in der `compose.yml` → **Step 3 – YAML-Syntax prüfen**, weil Tabs in YAML ungültig sind. `docker compose config --quiet` scheitert dann.
2. `html/` nie committet → **Step 7 – Inhalt prüfen**, weil `html/index.html` im Repo fehlt und nichts verglichen werden kann.
3. `"8085:8080"` → **Step 6 – Erreichbarkeit**. Port 8085 wird zwar veröffentlicht, Docker leitet aber auf Container-Port 8080 weiter. nginx lauscht im Container auf Port 80, also antwortet niemand.
4. `compose.yml` in einem Unterordner → **Step 2 – Compose-Datei suchen**. Der Step sucht nur im Hauptordner nach `compose.yaml`, `compose.yml`, `docker-compose.yaml` oder `docker-compose.yml`.

Hinweis: Step 5 startet nur den Service, der Port 8085 veröffentlicht (bei mir `web`). Uptime Kuma wird in der CI nicht gestartet.

**8. Warum `diff` statt nur Port-Check?**
Ein Webserver kann auf Port 8085 antworten und trotzdem die falsche Seite ausliefern, z. B. seine Standardseite, wenn das Volume falsch eingebunden ist. In Phase 5 wechseln wir auf Apache. Apache hat ein anderes Webroot als nginx (`/usr/local/apache2/htdocs/` statt `/usr/share/nginx/html`). Wird der Pfad nicht angepasst, liefert Apache „It works!“ statt meiner Seite aus. Der Port-Check wäre grün, nur der `diff` merkt den Fehler. Ein guter Test prüft also das Ergebnis, das der Kunde erwartet, und nicht nur, ob irgendetwas läuft.

**9. Wozu Step 10 Aufräumen?**
Step 10 führt `docker compose down -v` aus. Das stoppt und entfernt Container, Netzwerke und (wegen `-v`) auch die Volumes und sorgt für ein sauberes Ende. Bei GitHub-Runnern ist das nicht zwingend, weil die VM sowieso gelöscht wird. Bei selbst gehosteten Runnern, die wiederverwendet werden, würden sonst aber Container liegen bleiben und Ports wie 8085 blockieren. Der nächste Lauf würde dann scheitern. Aufräumen gehört zu jedem sauberen Test, und `if: always()` sorgt dafür, dass es auch nach einem Fehler passiert.

## Teil C: Einordnen

**10. CI, Delivery oder Deployment?**
Das ist **Continuous Integration**. Jede Änderung wird automatisch gebaut, gestartet und getestet. Es wird aber kein fertiges Artefakt bereitgestellt, z. B. ein Image in einer Registry, also keine Continuous Delivery. Es wird auch nichts automatisch auf den Server ausgerollt, also kein Continuous Deployment. Den LXC aktualisiere ich selbst mit `git pull` und `docker compose up -d`.

**11. Vorteile für die TechNova GmbH**
- Der Test läuft automatisch bei jeder Änderung und kann nicht vergessen werden.
- Er läuft immer gleich in einer sauberen Umgebung. „Bei mir geht's“ zählt nicht mehr, Fehler, die nur auf einem Rechner zufällig funktionieren, fallen auf.
- Fehler werden vor dem Betrieb entdeckt, nicht erst vom Kunden.
- Das Ergebnis ist dokumentiert und für alle sichtbar (Haken am Commit, Abzeichen, Logs).

## Teil D (Experten): DoD-Abzeichen

**12. Warum läuft `dod-3-branch-dev.yml` auf `dev`?**
Ein Abzeichen zeigt immer den letzten Lauf auf einem bestimmten Branch. DoD 3 soll prüfen, ob es den Branch `dev` gibt und er korrekt ist. Die Prüfung muss also bei einem Push auf `dev` laufen, und das Abzeichen muss mit `?branch=dev` genau diese Läufe anzeigen. Solange `dev` nicht existiert, gibt es keinen Lauf, und das Abzeichen ist grau. Die anderen DoD-Prüfungen (Repo, Apache auf `main`, Pull Request) betreffen den Betriebsstand und laufen deshalb auf `main`.

**13. Welche Infos braucht `dod-5-pull-request.sh`?**
- den zuletzt gemergten Pull Request von `dev` nach `main`
- seinen Merge-Zeitpunkt (`merged_at`)
- seinen letzten Commit (`head.sha`)
- das Ergebnis des Checks „Compose-Stack testen (Port 8085)“ für genau diesen Commit, mit Abschlusszeitpunkt (`completed_at`)

DoD 5 ist nur grün, wenn dieser Check erfolgreich war und fertig war, bevor gemergt wurde (`completed_at` < `merged_at`).
