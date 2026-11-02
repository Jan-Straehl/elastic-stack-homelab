# Antworten zur Bewerberaufgabe Elastic Stack

## Teil 1 – Cluster mit drei Nodes und Frozen Tier

### Warum drei master-fähige Nodes statt zwei?

Elasticsearch benötigt eine Mehrheit der stimmberechtigten Master-Nodes (Quorum), um einen Master zu wählen.

- **2 Nodes:** Mehrheit 2 – beim Ausfall eines Nodes keine Mehrheit mehr.
- **3 Nodes:** Mehrheit 2 – ein Node darf ausfallen, ohne dass die Mehrheit verloren geht.

Drei master-fähige Nodes erhöhen damit die Ausfallsicherheit des Clusters.

### Welche Host-Einstellungen benötigt Elasticsearch?

Unter Linux muss insbesondere `vm.max_map_count` ausreichend hoch sein, damit Elasticsearch genügend virtuelle Speicherbereiche verwalten kann. Auf dem Host wurde der Wert auf `1048576` gesetzt; die konkreten Befehle stehen im Abschnitt **Host-Vorbereitung**.

Weitere Voraussetzungen sind ausreichend RAM und CPU, Docker Engine und Docker Compose, passende Dateiberechtigungen und Container-Limits sowie die Erreichbarkeit der Nodes untereinander.

### Welche Rolle spielt der Shared Cache auf dem Frozen Node?

Der Shared Cache speichert häufig benötigte Datenblöcke aus Searchable Snapshots lokal zwischen. Dadurch müssen diese Blöcke nicht bei jeder Suchabfrage erneut aus dem Snapshot-Repository geladen werden.

**Konfiguration für `es03` in Docker Compose:**

```yaml
- xpack.searchable.snapshot.shared_cache.size=2GB
```

Der Shared Cache wurde für die kleine Testumgebung auf **2 GB** begrenzt, um die zu grosse automatische Cache-Reservierung zu vermeiden. In Produktion würde ich ihn abhängig von Datenmenge, Suchlast und verfügbarem SSD-Speicher dimensionieren. Das ursprüngliche Startproblem mit `es03` ist unter **Aufgetretene Probleme und Lösungen** dokumentiert.

## Teil 2 – Snapshots und Lifecycle bis Frozen

### Warum wurde MinIO als Snapshot-Speicher gewählt?

Als S3-kompatibler Snapshot-Speicher wurde MinIO gewählt, da es sich einfach in Docker Compose integrieren lässt und die S3-Schnittstelle von Elasticsearch unterstützt.

## Teil 3 – Fleet Server und Elastic Agent

### Wie funktioniert die TLS-Vertrauenskette zwischen Agent, Fleet Server und Elasticsearch?

Alle Komponenten verwenden Zertifikate der bestehenden Elastic-CA.

**Elastic Agent → Fleet Server**
- Verbindung über HTTPS auf Port 8220.
- Der Agent überprüft das Fleet-Server-Zertifikat anhand der CA-Datei.
- Die Registrierung erfolgt über einen Enrollment Token.

**Fleet Server → Elasticsearch**
- Verbindung über `https://es01:9200`.
- Überprüfung des Elasticsearch-Zertifikats anhand der CA.
- Authentifizierung mittels Service Token.

**Elastic Agent → Elasticsearch**
- Die gesammelten Daten werden über den konfigurierten Fleet Output an Elasticsearch übertragen.
- Die Zertifikatsprüfung erfolgt ebenfalls über die Elastic-CA.

**CA-Datei im Container:**

```text
/usr/share/elastic-agent/certs/ca/ca.crt
```

Dadurch kann auf das Deaktivieren der TLS-Zertifikatsprüfung verzichtet werden.

## Teil 4 – Fleet Policy ausrollen

### Wie würdest du eine Policy-Änderung auf 500 Agents ausrollen, ohne alle gleichzeitig zu treffen?

Bei 500 Agents würde ich Policy-Änderungen gestaffelt ausrollen:

1. Neue Konfiguration zunächst auf wenigen Test-Agents prüfen.
2. Rollout schrittweise auf weitere Agent-Gruppen erweitern.
3. Nach jeder Phase Agent-Status, Fehlermeldungen und Ressourcenverbrauch überwachen.
4. Bei Problemen den Rollout stoppen und die vorherige Konfiguration wiederherstellen.

Dafür würde ich separate Agent Policies für die jeweiligen Rollout-Gruppen verwenden.

So lassen sich Fehler früh erkennen und Auswirkungen auf die gesamte Infrastruktur reduzieren.

## Teil 5 – Konfiguration verstehen und abfragen

### 1. Konfigurationsquellen

Elasticsearch unterstützt verschiedene Konfigurationsquellen.

**Im Projekt verwendete Quellen:**

| Quelle | Verwendung |
|---|---|
| `docker-compose.yml` | Node-Rollen, Clustername, TLS, S3-Einstellungen |
| `.env` | Passwörter, Versionen, Container-Ressourcen |
| `elasticsearch.yml` | Nicht separat gemountet |
| JVM-Optionen | Automatische Heap-Dimensionierung |
| Elasticsearch-Keystore | S3-Zugangsdaten für MinIO |
| Cluster-Settings-API | Dynamische Einstellungen zur Laufzeit |

**Beispiel Docker Compose:**

```yaml
environment:
  - cluster.name=docker-cluster
  - xpack.security.enabled=true
```

**Elasticsearch-Keystore:**

Die MinIO-Zugangsdaten werden über folgende Secure Settings verwaltet:

```text
s3.client.default.access_key
s3.client.default.secret_key
```

Secure Settings können bei unterstützten Änderungen ohne Neustart neu geladen werden:

```http
POST /_nodes/reload_secure_settings
```

### 2. Effektive Node-Einstellungen ermitteln

Die tatsächlich aktiven Einstellungen wurden über die Elasticsearch-API in Kibana Dev Tools abgefragt.

**Node-Einstellungen:**

```http
GET /_nodes/es01/settings?flat_settings=true
```

**Cluster-Einstellungen inklusive Defaults:**

```http
GET /_cluster/settings?include_defaults=true&flat_settings=true
```

**Ermittelte Einstellungen auf es01:**
- Clustername: `docker-cluster`
- Security: aktiviert
- TLS: aktiviert
- S3-Endpunkt: `http://minio:9000`
- S3-Region: `us-east-1`
- S3 Path-Style-Access: `true`
- Node-Rollen: master, data_hot, data_content, ingest, transform

Die Node-API zeigt die explizit gesetzten Node-Einstellungen. Die Cluster-Settings-API liefert zusätzlich die Standardwerte für Cluster-Einstellungen.

### 3. Priorität der Konfigurationsquellen

Bei dynamischen Cluster-Einstellungen gilt folgende Priorität:

1. Transient Cluster Settings
2. Persistent Cluster Settings
3. `elasticsearch.yml` bzw. entsprechende Node-Konfiguration
4. Standardwerte (Defaults)

**Prioritätstest:**

Zunächst wurde ein persistenter Wert von 40 MB/s gesetzt:

```http
PUT /_cluster/settings
{
  "persistent": {
    "indices.recovery.max_bytes_per_sec": "40mb"
  }
}

```

Anschliessend wurde dieselbe Einstellung transient auf 50 MB/s gesetzt:

```http
PUT /_cluster/settings
{
  "transient": {
    "indices.recovery.max_bytes_per_sec": "50mb"
  }
}
```

**Überprüfung:**

```http
GET /_cluster/settings?flat_settings=true
```

**Ergebnis:**
- Persistent: 40 MB/s
- Transient: 50 MB/s
- Effektiver Wert: 50 MB/s

Damit wurde nachgewiesen, dass die transiente Einstellung die persistente Einstellung übersteuert.

Dynamische Cluster-Settings besitzen zudem eine höhere Priorität als entsprechende Einstellungen in `elasticsearch.yml`.

**Testkonfiguration zurücksetzen:**

```http
PUT /_cluster/settings
{
  "persistent": {
    "indices.recovery.max_bytes_per_sec": null
  },
  "transient": {
    "indices.recovery.max_bytes_per_sec": null
  }
}
```

Nach dem Test wurden die Einstellungen auf ihre Standardwerte zurückgesetzt.

### 4. JVM-Heap und Node-Rollen ermitteln

Die tatsächlich verwendeten JVM-Heap-Grössen und Node-Rollen wurden ohne Einsicht in Docker Compose abgefragt.

**JVM-Heap:**

```http
GET /_nodes/stats/jvm?filter_path=nodes.*.name,nodes.*.jvm.mem.heap_max_in_bytes
```

**Ergebnis:**

| Node | JVM-Heap |
|---|---|
| es01 | 512 MiB |
| es02 | 512 MiB |
| es03 | 512 MiB |

Jeder Node verwendet tatsächlich 536'870'912 Bytes JVM-Heap.

**Node-Rollen:**

```http
GET /_nodes?filter_path=nodes.*.name,nodes.*.roles
```

**Ergebnis:**

| Node | Rollen |
|---|---|
| es01 | master, data_hot, data_content, ingest, transform |
| es02 | master, data_hot, data_content, ingest, transform |
| es03 | master, data_frozen |

Die tatsächlichen Laufzeiteinstellungen wurden direkt über Elasticsearch ermittelt und bestätigt.

### 5. Unterschiedliche Schreibweisen der Umgebungsvariablen

Elasticsearch verwendet in Docker Compose Einstellungen mit Punktnotation.

**Elasticsearch:**

```yaml
- xpack.security.enabled=true
```

Kibana verwendet dagegen Umgebungsvariablen in Grossbuchstaben mit Unterstrichen.

**Kibana:**

```yaml
- ELASTICSEARCH_HOSTS=https://es01:9200
```

Fleet Server verwendet ebenfalls grossgeschriebene Umgebungsvariablen mit Unterstrichen, allerdings mit eigenen Fleet-spezifischen Namen.

**Fleet Server:**

```yaml
- FLEET_SERVER_ELASTICSEARCH_HOST=https://es01:9200
```

Die unterschiedlichen Schreibweisen werden beim Start der jeweiligen Container durch die Anwendungen interpretiert.

