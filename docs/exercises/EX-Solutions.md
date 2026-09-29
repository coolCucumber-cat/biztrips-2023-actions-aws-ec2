# EX-Solutions – Musterlösungen zu den Reflexionsfragen

---

## EX-03 – Deployment auf AWS ECS (Fargate)

### Frage 1

**Welche Aufgaben, die in EX-01 der SSH-Deploy-Schritt (`rsync` + `sudo systemctl reload nginx`) manuell erledigt hat, übernimmt in dieser Übung die ECS-Kontrollebene automatisch?**

In EX-01 hat der Deploy-Schritt per SSH manuell:
- Dateien auf den Server kopiert (`rsync`)
- nginx neu geladen (`sudo systemctl reload nginx`)
- Den Prozess am Leben gehalten (kein automatischer Neustart bei Absturz)

In ECS übernimmt die Kontrollebene:
- **Scheduling**: ECS entscheidet, auf welcher Infrastruktur der Container läuft
- **Start/Stopp**: Container starten automatisch aus der Task Definition heraus, kein SSH nötig
- **Self-Healing**: Crasht ein Container oder besteht er den Health-Check nicht, ersetzt ECS ihn automatisch
- **Rolling Deployment**: Neue Tasks starten, bevor alte gestoppt werden — kein manuelles Eingreifen, kein Ausfall

---

### Frage 2

**Warum ist eine über OIDC bezogene, kurzlebige IAM-Rolle sicherer als ein dauerhaft in GitHub Secrets hinterlegter AWS-Access-Key?**

| OIDC | Dauerhafter Key |
|------|-----------------|
| Token gilt nur für **einen Job-Run** (Minuten) | Key ist **permanent gültig** bis zur manuellen Rotation |
| Kein Secret gespeichert — GitHub tauscht nur ein kurzlebiges Token aus | Key liegt dauerhaft in GitHub Secrets gespeichert |
| Wird ein Token geleakt: er ist bereits abgelaufen | Wird ein Key geleakt: Angreifer hat Zugriff bis zum Widerruf |
| Scope auf exaktes Repo + Branch einschränkbar | Key hat die vollen Rechte der zugehörigen IAM-Policy |

Der entscheidende Vorteil: Bei OIDC existiert **kein langlebiges Geheimnis**, das gestohlen werden könnte. GitHub beweist seine Identität mit einem signierten JWT-Token, AWS prüft die Signatur und stellt ein kurzlebiges Session-Token aus — ohne dass jemals ein Access Key irgendwo gespeichert werden muss.

---

### Frage 3

**Was passiert mit den zwei laufenden Tasks eines Services, wenn ein `update-service` mit neuer Task-Definition ausgelöst wird — und warum ist das ein Vorteil gegenüber dem Single-Server-Deployment aus EX-01?**

ECS führt ein **Rolling Deployment** durch:

1. Neue Tasks mit der neuen Task-Definition (= neuem Image) werden gestartet
2. ECS wartet, bis die neuen Tasks den Health-Check des ALB bestehen (HTTP 200 auf `/`)
3. Erst dann werden die alten Tasks gestoppt

Während des gesamten Vorgangs leitet der ALB Traffic immer nur auf **gesunde Tasks** — entweder alte oder neue, aber nie auf Tasks, die noch hochfahren oder bereits abgeschaltet werden.

Vorteil gegenüber EX-01: In EX-01 gab es genau **einen Server**. Während `rsync` lief und nginx neu geladen wurde, war die App kurz in einem inkonsistenten Zustand oder kurz nicht erreichbar. Beim Rolling Deployment gibt es **keinen Ausfall** und keine Downtime.

---

### Frage 4

**Warum braucht die Target Group in Schritt 4 den Typ `ip` statt `instance`?**

Bei EC2-basierten Deployments (EX-01) hat jede Instanz eine feste ID (`i-xxxx`) — der ALB kann sie direkt als Ziel registrieren (Typ `instance`).

Fargate-Tasks laufen **ohne sichtbare EC2-Instanz**. Sie bekommen eine private IP-Adresse über eine ENI (Elastic Network Interface) — diese IP wechselt bei jedem Task-Neustart. Es gibt keine Instance-ID, auf die der ALB zeigen könnte. Der ALB muss daher direkt gegen die **IP-Adresse** des Tasks routen (Typ `ip`), die ECS beim Start automatisch in der Target Group registriert und beim Stopp wieder entfernt.
