# Konzept: Kuma → Upptime Migration

**Stand:** 2026-08-11 · **Status:** Entwurf zur Diskussion
**Ausgangslage:** Uptime-Monitoring läuft aktuell auf Uptime Kuma (`uptime.beta.formfollowsyou.com`, Container `95b9cc90cdd9` auf `beta`). Ziel: externe, reliable Überwachung via **Upptime** (GitHub Actions / Issues / Pages), ohne eigenen Server.

---

## 1. Ist-Zustand (aus Kuma-DB-Export `kuma-export/kuma.db`)

**DB-Kennzahlen:**
- 35 Monitore (alle aktiv), 7.232.009 Heartbeats, DB ~1 GB, Retention 180 Tage, Daten seit Feb 2026
- 1 Notification-Provider, 1 Status-Page ("Buildplace"), 2 Gruppen, 9 Tags
- Server-Zeitzone `Europe/Berlin`

### 1.1 Monitore nach Typ

| # | Typ | Anzahl | Monitore |
|---|-----|--------|----------|
| HTTP | `http` | 20 | Docker Registry, Thermal Comfort, Buildplace, Geodata (dev/prod), Geoserver, RocketChat, Buildplace Dev, Hal-Plan, XPlan Web/API/Services (dev+hal+prod = 9), Grafana, Thermal Comfort Prod |
| Keyword | `keyword` | 8 | Storage Proxy (dev/hal/prod), Bucket Direct (dev/hal/prod), Bucket Direct Backup (hal/prod) – Logik: **Status 403 + Keyword im Body** |
| Ping | `ping` | 4 | Load Balancer (dev/hal/prod/mgmt) – ICMP auf öffentliche IPs (Hetzer) |
| Group | `group` | 3 | XPlan IaC Dev/Hal/Prod – nur Parent-Aggregationen (Kinder = 23–25, 27–29, 31–33) |

**Intervalle:** 20 s (Buildplace, Buildplace Dev, Hal-Plan) · 60 s (alle übrigen) · 87000 s (Gruppen)
**Aktueller Zustand:** alle 35 Monitore UP, Uptime 99,4–100 %. Meiste Downtimes: Buildplace (99,75 %), Geodata (99,70 %), Storage Proxies (99,43–99,70 %), Grafana (99,85 %).

### 1.2 Alerting (wichtig!)

- **1 Notification:** "My Rocket.Chat Alert" → Webhook `rocketchat.serve.buildplace.io/hooks/…` → Kanal `#Server`, Absender `uptime-kuma`.
- **27 von 35 Monitoren** sind mit dieser Notification verknüpft.
- **8 Monitore haben KEINEN Alert:** Storage Proxy (dev/hal/prod) + Bucket Direct (dev/hal/prod) + Bucket Direct Backup (hal/prod). → Diese sind heute stumm, obwohl sie die meiste Downtime haben (99,4–99,8 %).

### 1.3 Status-Page & sonstiges
- Status-Page "Buildplace" (published), 1 Maintenance-Eintrag vorhanden, kein TLS-Expiry-Alerting aktiv, kein API-Key.

---

## 2. Zielbild: Upptime

```
┌─────────────────────── GitHub (public Repo) ───────────────────────┐
│  Uptime CI (alle 5 min)  →  prüft 32 Endpoints von GitHub-Runnern  │
│  Issues = Incidents      →  automatisch öffnen/schließen           │
│  Notification            →  Rocket.Chat (Slack-kompatibler Webhook)│
│  GitHub Pages            →  Status-Page (z.B. status.buildplace.io)│
│  api/*.json              →  öffentliche JSON-Endpunkte pro Dienst   │
└─────────────────────────────────────────────────────────────────────┘
  ✔ externe Checks (unabhängig von beta)   ✔ kein Server/Wartung
  ✔ kostenlos (public Repo)                ✔ Git-Historie = Audit-Trail
```

**Kernvorteil gegenüber Kuma:** Checks laufen auf GitHub-Infrastruktur, nicht auf dem eigenen Server → Ausfälle von `beta` selbst werden erkannt.

---

## 3. Mapping Kuma → Upptime (32 echte Endpoints)

Gruppen-Monitore (21/26/30) entfallen → Kinder werden einzeln überwacht, die Status-Page aggregiert automatisch.

| Kuma-Monitor | Kuma-Typ | Upptime-Config (Entwurf) |
|---|---|---|
| FFY Docker Registry, Thermal Comfort, Thermal Comfort Prod, Geoserver, RocketChat, Grafana | http 200–299 | `url: <url>` (Standard-Statuscodes) |
| Buildplace, Buildplace Dev, Hal-Plan | http 200–299, 20 s | `url: <url>`, `maxResponseTime: 10000` |
| Geodata (dev/prod), XPlan Web/API/Services (dev/hal/prod) | http 200–299 | `url: <url>` |
| Storage Proxy (dev/hal/prod) | keyword: 403 + `nbg1`/`fsn1` | `expectedStatusCodes: [403]` + `__dangerous__body_down_if_text_missing: nbg1` (bzw. fsn1) |
| Bucket Direct (dev/hal/prod), Bucket Direct Backup (hal/prod) | keyword: 403 + `AccessDenied` | `expectedStatusCodes: [403]` + `__dangerous__body_down_if_text_missing: AccessDenied` |
| Load Balancer (dev/hal/prod/mgmt) | ping (ICMP) | **Variante A:** `type: globalping` + `check: tcp-ping` + `location: berlin` (echter Ping, von außen) · **Variante B:** `check: tcp-ping` + `port: 443` (direkt vom Runner) |
| XPlan IaC Gruppen (21/26/30) | group | entfällt (Kinder ersetzen) |

**Hinweis Keyword-Logik:** Kuma = "up wenn Status 403 UND Keyword vorhanden". Upptime bildet das 1:1 ab: `expectedStatusCodes: [403]` + `body_down_if_text_missing`.

---

## 4. Alerting bei Upptime

Upptime hat **keinen nativen Rocket.Chat-Provider** (verifiziert im Quellcode). Drei Optionen:

1. **✅ Empfohlen – Slack-Provider auf Rocket.Chat-Webhook:** Rocket.Chat-Incoming-Webhooks akzeptieren das Slack-Payload-Format (`{text: …}`). Der Slack-Provider von Upptime sendet genau das.
   ```
   NOTIFICATION_SLACK            = true
   NOTIFICATION_SLACK_WEBHOOK_URL = https://rocketchat.serve.buildplace.io/hooks/68d6… (bestehende URL)
   ```
   → nur 1 Secret, kein Zusatzcode. **Muss mit einem echten Test verifiziert werden** (Dienst kurz stoppen).
2. **Custom Webhook + Mini-Adapter:** Upptime sendet `{data:{message:…}}`, Rocket.Chat erwartet `{text:…}` → kleiner Adapter nötig (z.B. Cloudflare Worker / Mini-Script auf `beta`), der das Format übersetzt. Mehr bewegliche Teile.
3. **Zusätzlicher Kanal als Backup** (Telegram, E-Mail/SMTP): unabhängig von Rocket.Chat, falls Rocket.Chat selbst down ist. *Empfehlung: zusätzlich mindestens einen zweiten Kanal einrichten – Rocket.Chat wird selbst überwacht und ist bei Rocket.Chat-Ausfall blind.*

**Verbesserung gegenüber heute:** In Upptime bekommen **alle 32 Endpoints** Alerts – auch die 8 Storage/Bucket-Monitore, die in Kuma stumm sind.

---

## 5. Was gebaut / gemacht werden muss (Aufgabenliste)

### Phase 0 – Entscheidungen
- [ ] **GitHub-Ziel festlegen:** Repo z.B. `FormFollowsYou/uptime` oder `buildplace/uptime`, **public** (Status-Page + API öffentlich, 0 € Actions-Kosten). Privat = 2.000 min/Monat reichen nicht bei 5-min-Checks.
- [ ] **Status-Page-Domain:** z.B. `status.buildplace.io` oder `uptime.formfollowsyou.com` (CNAME auf `user.github.io`) – oder erstmal `user.github.io/uptime/`.

### Phase 1 – Repo & Basis
- [ ] Repo aus Template `upptime/upptime` anlegen (`gh repo create … --template upptime/upptime` – `gh` ist lokal installiert).
- [ ] `.upptimerc.yml` schreiben: `owner`, `repo`, **32 Sites** (Mapping oben), `status-website`, `workflowSchedule` (Standard = alle 5 min).
- [ ] Secrets anlegen (Repo → Settings → Secrets → Actions):
  - `GH_PAT` (Personal Access Token mit `repo`-Rechten, damit der Bot committen kann – bei public Repo optional)
  - `NOTIFICATION_SLACK` + `NOTIFICATION_SLACK_WEBHOOK_URL` (Rocket.Chat)
  - optional `SECRET_SITE` für URLs, die nicht ins Repo sollen
- [ ] Ersten Uptime-Run abwarten, Status-Page-Check.

### Phase 2 – Test & Parallelbetrieb
- [ ] **Alert-Test:** einen Dienst stoppen → Rocket.Chat-Nachricht innerhalb ≤5 min erwartet → wieder starten → "Back up"-Meldung.
- [ ] 1–2 Wochen **parallel** zu Kuma laufen lassen (Kuma bleibt Quelle, Upptime als zweite Meinung / "externes Auge").
- [ ] Downtimes von Upptime vs. Kuma abgleichen (v.a. bei Storage-Proxies: 403+Keyword-Logik korrekt?).

### Phase 3 – Umstellung
- [ ] Kuma-Monitore nach und nach deaktivieren (nicht löschen – Historie behalten).
- [ ] Optional: Kuma-Container stoppen / nur noch bei Bedarf starten, DNS/Status-Link auf neue Status-Page umbiegen.
- [ ] Kuma-DB + `kuma-export/` archivieren (Backup).

---

## 6. Trade-offs & Risiken (ehrlich benannt)

| Thema | Kuma (heute) | Upptime | Bewertung |
|---|---|---|---|
| Check-Intervall | 20–60 s | **min. 5 min** (GitHub-Cron-Limit) | Downtimes werden später erkannt; für kritische Dienste ggf. Ergänzung nötig (z.B. Globalping-Frequenz oder zweites Tool) |
| Standort der Checks | eigener Server (beta) | GitHub-Runner (externe IPs) | **genau das gewünschte Ziel** |
| Private/interne Dienste | ✔ (aus dem Netz) | ✖ nur öffentlich erreichbare | hier nicht relevant – alle URLs sind öffentlich |
| Retries pro Check | ✔ maxretries | ✖ 1 Versuch = 1 Ergebnis | mehr Fehlalarme möglich (kurze Aussetzer) |
| Historie | 7,2 Mio Heartbeats (180 Tage) | startet **bei 0**, wächst in Git | keine 1:1-Migration der Historie; Statistik beginnt neu |
| Keyword-Checks | ✔ | ✔ (`body_down_if_text_missing`) | abbildbar |
| ICMP-Ping | ✔ nativ | nur via Globalping (kostenloses Limit: 250 Tests/h ohne Token) | Load Balancer alternativ als TCP:443-Check |
| Gruppen/Abhängigkeiten | ✔ Parent-Kinder | ✖ | entfällt (Status-Page aggregiert) |
| Kosten | Server + Wartung | 0 € (public Repo) | + |
| Rocket.Chat-Alert | ✔ nativ | über Slack-kompatiblen Webhook (Test nötig) | + |

**Größter praktischer Nachteil:** 5-Minuten-Intervall + keine Retries → kurze Aussetzer (<5 min) können als Downtime zählen, lange Ausfälle werden bis zu 5 min später gemeldet. Für "reliable externe Überwachung" in der Regel akzeptabel.

---

## 7. Offene Fragen an dich
1. GitHub-Ziel: Organisation `FormFollowsYou`? Repo-Name? Public ok?
2. Status-Page-Domain: `status.buildplace.io`? Sonst `uptime.formfollowsyou.com`?
3. Rocket.Chat bleibt der einzige Alarm-Kanal, oder zusätzlich Telegram/E-Mail?
4. Sind die 20-s-Kritisch-Monitore (Buildplace-Familie) wirklich unkritisch bei 5 min Takt, oder brauchen wir einen Zusatz (Globalping 1-min, zweites Tool)?
5. Sollen die bisher stummen Storage/Bucket-Monitore künftig **mit** Alert laufen (Empfehlung: ja)?

---

## 8. Nächster Schritt
Nach deinen Antworten aus Abschnitt 7:
1. Repo anlegen + `.upptimerc.yml` komplett ausformulieren (alle 28 Endpoints, mit den echten URLs aus dieser Analyse).
2. Secrets setzen, Status-Page-Domain, ersten Alert-Test durchführen.
3. Migrations-Checkliste Phase 2/3 durcharbeiten.
