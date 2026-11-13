# Hetzner Cloud GitHub Runner Workflows

Dieses Repository enthält drei GitHub Actions Workflows zum Verwalten selbst gehosteter GitHub Runner auf Hetzner Cloud:

- `create-hetzner-runners.yml` – neue Runner erstellen
- `rotate-hetzner-runners.yml` – bestehende Runner austauschen
- `delete-hetzner-runners.yml` – Runner entfernen

Die Workflows verwenden das Action-Plugin `wydler/hcloud-self-hosted-github-runner` und arbeiten mit GitHub App Tokens sowie Hetzner Cloud API-Zugriff.

## Voraussetzungen

Folgende GitHub Secrets müssen im Repository oder der Organisation hinterlegt sein:

- `HCLOUD_TOKEN_GHSHR_RW`: Hetzner Cloud Token für das entsprechende Projekt in Hetzner Cloud, mit den notwendigen Rechten zur Verwaltung der Server.
- `GH_APP_SHRHC_ID`: ID der GitHub App, die die erforderlichen Rechte besitzt, um Runner im Repository oder in der Organisation zu verwalten.
- `GH_APP_SHRHC_PRIVATE_KEY`: Privater Schlüssel der GitHub App, der zusammen mit `GH_APP_SHRHC_ID` für die GitHub Runner-Verwaltung verwendet wird.

Für die GitHub App gelten je nach Scope unterschiedliche Rechte:

- Für `org`: erforderlich ist das Recht `Self-hosted runners` mit `Read and write`.
- Für `repo`: erforderlich ist das Recht `Administration` mit `Read and write`.

Als Runner-Scope kann `repo` oder `org` gewählt werden. Runner-Gruppen gibt es nur bei Organisationen (`org`):

- `Default`
- `Daniel-Hetzner`

Die benötigten Runner-Gruppen müssen vor der Workflow-Ausführung manuell in den Einstellungen der GitHub-Organisation angelegt werden.

Bei Scope `repo` wird keine Runner-Gruppe verwendet.

Hinweis: Bei `hcloud_ssh_key` handelt es sich nicht um eine ID, sondern um den Namen des SSH-Schlüssels in Hetzner Cloud.

## Allgemeiner Ablauf

Die Workflows prüfen zunächst, ob die GitHub Runner-Umgebung korrekt konfiguriert ist, holen dann passende Hetzner-Details wie Snapshot-ID und den Namen des SSH-Schlüssels und erstellen oder entfernen anschließend die Runner in Matrix-Form.

## 1) Workflow: Create

Datei: `.github/workflows/create-hetzner-runners.yml`

### Zweck
Erstellt einen oder mehrere neue self-hosted GitHub Runner auf Hetzner Cloud.

### Trigger
Manuell per `workflow_dispatch`.

### Eingaben

| Input | Beschreibung | Standard |
|---|---|---|
| `location` | Hetzner-Region | `fsn1` |
| `os` | Betriebssystem-Snapshot | `ubuntu-24.04` |
| `servertype` | Hetzner-Servertyp | `cx33` |
| `runner_count` | Anzahl der neuen Runner | `2` |
| `runner_scope` | `repo` oder `org` | `org` |
| `runner_group` | Runner-Gruppe (nur bei Scope `org`; bei `repo` nicht verwendet) | `Default` |
| `runner_start_method` | Startmethode des Runners | `systemd` |
| `hcloud_ssh_key` | Name des SSH-Schlüssels in Hetzner Cloud | erforderlich |
| `runner_labels` | Zusätzliche Labels | leer |

### Ablauf
1. Ermittelt die aktuelle Snapshot-ID für das gewünschte Betriebssystem und die passende Architektur (`x86` oder `arm`).
2. Sucht anhand des angegebenen Namens nach dem passenden Hetzner SSH-Key.
3. Validiert `runner_count` und bricht ab, wenn die Anzahl größer als 5 ist.
4. Erstellt für jeden Runner einen GitHub App Token.
5. Nutzt `wydler/hcloud-self-hosted-github-runner` mit `mode: create`.
6. Fügt eine Zusammenfassung zum Step Summary hinzu.

### Wichtig
- `runner_count` muss eine positive Zahl sein.
- Es können maximal 5 Runner pro Workflow-Lauf erstellt werden (`MAX_RUNNER_COUNT: 5`); höhere Werte werden abgelehnt.
- Für `CAX`-Server wird automatisch `arm` statt `x86` verwendet.

## 2) Workflow: Rotate

Datei: `.github/workflows/rotate-hetzner-runners.yml`

### Zweck
Ersetzt bestehende Hetzner Runner durch neue, frische Runner.

### Trigger
Manuell per `workflow_dispatch`.

### Eingaben

| Input | Beschreibung | Standard |
|---|---|---|
| `confirm_rotation` | Bestätigung, dass alte Runner nach erfolgreichem Ersatz gelöscht werden | `false` |
| `location` | Hetzner-Region | `fsn1` |
| `os` | Betriebssystem-Snapshot | `ubuntu-24.04` |
| `server_type` | Hetzner-Servertyp | `cx33` |
| `runner_scope` | `repo` oder `org` | `org` |
| `runner_group` | Runner-Gruppe (nur bei Scope `org`; bei `repo` nicht verwendet) | `Default` |
| `runner_start_method` | Startmethode des Runners | `systemd` |
| `hcloud_ssh_key` | Name des SSH-Schlüssels in Hetzner Cloud | erforderlich |
| `runner_labels` | Zusätzliche Labels | leer |

### Ablauf
1. Prüft, dass `confirm_rotation` auf `true` gesetzt wurde.
2. Sammelt vorhandene, von diesem Projekt verwaltete Runner mit `server_id`-Labels.
3. Validiert, dass jede Runner-Server-Korrelation eindeutig ist.
4. Ermittelt automatisch die Anzahl der validierten alten Runner und erstellt genau so viele neue Runner.
5. Sucht den neuesten passenden Hetzner Snapshot und den SSH-Key anhand des Namens.
6. Erstellt neue Runner mit `mode: create`.
7. Wartet, bis die bisherigen Runner idle sind.
8. Löscht die alten Runner mit `mode: delete`.
9. Zusammenfassung der Rotation im GitHub Step Summary.

### Wichtig
- Rotation ist nur mit expliziter Bestätigung möglich.
- Jeder validierte, verwaltete Runner wird genau durch einen neuen Runner ersetzt.
- Alte Runner werden erst gelöscht, wenn sie nicht mehr aktiv sind.
- Der Workflow arbeitet mit einem Matrix-Pattern für neue und alte Runner.

## 3) Workflow: Delete

Datei: `.github/workflows/delete-hetzner-runners.yml`

### Zweck
Entfernt verwaltete Hetzner Runner aus GitHub und Hetzner Cloud.

### Trigger
Manuell per `workflow_dispatch`.

### Eingaben

| Input | Beschreibung | Standard |
|---|---|---|
| `runner_scope` | `repo` oder `org` | `org` |
| `runner_group` | Runner-Gruppe (nur bei Scope `org`; bei `repo` nicht verwendet) | `Default` |
| `runner_count` | Anzahl der zu löschenden Runner oder `all` für alle validierten Hetzner-Runner | `1` |

### Ablauf
1. Erstellt für den ausgewählten Scope ein GitHub App Token.
2. Lädt alle GitHub Runner per GitHub API.
3. Filtert nur jene Runner, die ein Hetzner-Server-Label `server_id=` tragen.
4. Validiert, dass Name, Server-ID und Zuordnung konsistent sind.
5. Prüft bei einer festen Anzahl, ob genügend validierte Runner verfügbar sind; bei `all` werden alle validierten Runner ausgewählt.
6. Wählt die Runner alphabetisch nach Runner-Namen aus und baut daraus eine Matrix mit `runner` und `server_id`.
7. Löscht über `wydler/hcloud-self-hosted-github-runner` mit `mode: delete` die Server und GitHub Runner.
8. Speichert Ergebnisse als Artefakte und fasst sie am Ende zusammen.

### Wichtig
- Nur durch das Projekt verwaltete Runner werden gelöscht.
- `runner_count` muss eine positive Ganzzahl oder `all` sein. Eine feste Anzahl darf die Anzahl der validierten, verfügbaren Runner nicht überschreiten.
- Ein Runner wird nur entfernt, wenn die Server- und GitHub-Zuordnung eindeutig ist.
- Der Workflow dokumentiert die Lösch-Ergebnisse als Artefakte.
