c# Checkliste — AWS Academy Lab starten, EC2/ECR/ECS einrichten, GitHub Secrets setzen

Für den kompletten Durchlauf der Pipeline (`test` → `build` → `deploy` → `docker` → `deploy-ecs`)
in einer frischen AWS-Academy-Learner-Lab-Session. Reihenfolge von oben nach unten abarbeiten.

---

## 0. Lab starten

- [ ] AWS Academy: **Start Lab**, warten bis der Punkt neben "AWS" grün ist
- [ ] *AWS Details → AWS CLI* öffnen — von dort kommen gleich `AWS_ACCESS_KEY_ID`,
      `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`
- [ ] AWS-Console öffnen, Region oben rechts auf **us-east-1 (N. Virginia)** stellen
      (Learner Lab ist darauf fixiert — `task-definition.json` und `deploy.yml` gehen
      ebenfalls von `us-east-1` aus)

> ⚠️ Diese Zugangsdaten laufen mit der Lab-Sitzung ab (spätestens nach 4h, oder
> beim nächsten Neustart der Session). Schritt 4 (Secrets setzen) muss dann erneut
> durchgeführt werden.

---

## 1. EC2-Instanz für das Frontend (EX-01)

- [ ] EC2 → **Launch instance**
  - Name: z. B. `biztrips-frontend`
  - AMI: Amazon Linux 2023 (Login-User `ec2-user`) **oder** Ubuntu (Login-User `ubuntu`)
  - Instance-Type: `t3.micro` (Learner-Lab-Limit)
  - Key Pair: **vockey** (Private Key `labsuser.pem` unter *AWS Details → Download PEM*)
  - Security Group: Inbound `SSH (22)` und `HTTP (80)`, Quelle `0.0.0.0/0`
- [ ] Public IPv4 DNS notieren (z. B. `ec2-3-91-12-34.compute-1.amazonaws.com`)
- [ ] Erreichbarkeit **vor** dem ersten Pipeline-Run testen:
  ```bash
  nc -z -v <public-dns> 22
  ssh -i labsuser.pem <user>@<public-dns> true && echo "ssh ok"
  ```
- [ ] nginx + rsync auf der Instanz installieren, `/var/www/biztrips` anlegen
      (Details: [EX-01](EX-01-deploy-AWS-EC2.md) Schritt 2–3, alternativ
      [EX-00](EX-00-Vorbereitungen-AWS-Linux.md) für Amazon-Linux-spezifische Befehle —
      **Achtung:** EX-00 nutzt `dnf`/Amazon Linux, das README `apt`/Ubuntu; je nach
      gewählter AMI den passenden Befehlssatz verwenden)
- [ ] `http://<public-dns>/` liefert eine Testseite, **bevor** die Pipeline angefasst wird

---

## 2. ECR-Repository + ECS-Cluster/Service (EX-03)

Alles in **us-east-1**, Ressourcennamen so wählen, dass sie zu den Namen im
Workflow passen (`ECR_REPOSITORY: biztrips`, `cluster: biztrips-cluster`,
`service: biztrips-service`, `family: biztrips` in `task-definition.json`).

### 2a. ECR-Repository anlegen

- [ ] `aws ecr create-repository --repository-name biztrips --region us-east-1`
- [ ] Repository-URI notieren: `aws ecr describe-repositories --repository-names biztrips --region us-east-1 --query "repositories[0].repositoryUri" --output text`

### 2b. ECS-Cluster anlegen

- [ ] `aws ecs create-cluster --cluster-name biztrips-cluster --region us-east-1`
- [ ] Status prüfen: `aws ecs describe-clusters --clusters biztrips-cluster --region us-east-1 --query "clusters[0].status" --output text` → `ACTIVE`

### 2c. VPC und Subnets ermitteln

- [ ] Default-VPC-ID notieren:
  ```bash
  aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
    --query "Vpcs[0].VpcId" --output text --region us-east-1
  ```
- [ ] Mindestens zwei Subnet-IDs aus der Default-VPC notieren (für ALB und ECS-Service):
  ```bash
  aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>" \
    --query "Subnets[*].SubnetId" --output text --region us-east-1
  ```

### 2d. Security Groups anlegen

- [ ] Security Group für den **ALB** anlegen (eingehend Port 80, `0.0.0.0/0`):
  ```bash
  aws ec2 create-security-group --group-name biztrips-alb-sg \
    --description "ALB biztrips" --vpc-id <vpc-id> --region us-east-1
  aws ec2 authorize-security-group-ingress \
    --group-id <alb-sg-id> --protocol tcp --port 80 --cidr 0.0.0.0/0 --region us-east-1
  ```
- [ ] Security Group für die **Fargate-Tasks** anlegen (eingehend Port 80 **nur von ALB-SG**):
  ```bash
  aws ec2 create-security-group --group-name biztrips-tasks-sg \
    --description "ECS Tasks biztrips" --vpc-id <vpc-id> --region us-east-1
  aws ec2 authorize-security-group-ingress \
    --group-id <tasks-sg-id> --protocol tcp --port 80 \
    --source-group <alb-sg-id> --region us-east-1
  ```

### 2e. Target Group und Application Load Balancer anlegen

- [ ] Target Group vom Typ `ip` anlegen (Health-Check `/`):
  ```bash
  aws elbv2 create-target-group --name biztrips-tg \
    --protocol HTTP --port 80 --vpc-id <vpc-id> \
    --target-type ip --health-check-path / --region us-east-1
  ```
  Target-Group-ARN notieren.
- [ ] Application Load Balancer in mindestens zwei Subnets anlegen:
  ```bash
  aws elbv2 create-load-balancer --name biztrips-alb \
    --subnets <subnet-1> <subnet-2> --security-groups <alb-sg-id> --region us-east-1
  ```
  ALB-ARN und ALB-DNS-Namen notieren.
- [ ] Listener Port 80 → Target Group anlegen:
  ```bash
  aws elbv2 create-listener --load-balancer-arn <alb-arn> \
    --protocol HTTP --port 80 \
    --default-actions Type=forward,TargetGroupArn=<tg-arn> --region us-east-1
  ```

### 2f. task-definition.json anpassen

- [ ] `executionRoleArn` → `arn:aws:iam::123456789012:role/LabRole`
      (Platzhalter-Account-ID bleibt so — der `deploy-ecs`-Job ersetzt sie zur Laufzeit
      automatisch via `aws sts get-caller-identity`, kein manuelles Anpassen nötig)
- [ ] `awslogs-region` und ECR-Image-URI sind bereits auf `us-east-1` gesetzt
- [ ] `"awslogs-create-group": "true"` muss vorhanden sein — ohne dieses Flag
      schlägt der Task-Start fehl, wenn die Log-Gruppe noch nicht existiert

### 2g. Task Definition registrieren und ECS-Service anlegen

- [ ] Task Definition registrieren:
  ```bash
  aws ecs register-task-definition --cli-input-json file://task-definition.json --region us-east-1
  ```
- [ ] ECS-Service anlegen (`desired-count 2` = zwei Tasks für Redundanz):
  ```bash
  aws ecs create-service \
    --cluster biztrips-cluster --service-name biztrips-service \
    --task-definition biztrips --desired-count 2 --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[<subnet-1>,<subnet-2>],securityGroups=[<tasks-sg-id>],assignPublicIp=ENABLED}" \
    --load-balancers "targetGroupArn=<tg-arn>,containerName=biztrips,containerPort=80" \
    --region us-east-1
  ```

### 2h. Verifizierung

- [ ] Tasks laufen: `aws ecs list-tasks --cluster biztrips-cluster --service-name biztrips-service --region us-east-1` → zwei Task-ARNs sichtbar
- [ ] Task-Status: `aws ecs describe-tasks --cluster biztrips-cluster --tasks <task-arn> --region us-east-1 --query "tasks[0].lastStatus" --output text` → `RUNNING`
- [ ] ALB erreichbar: `curl -I http://<alb-dns-name>/` → HTTP 200 (erst nach dem ersten `deploy-ecs`-Pipeline-Durchlauf, wenn ein Image in ECR vorhanden ist)

---

## 3. GitHub Secrets & Variables setzen

### 3a. Environment `production` (nur für den `deploy`-Job, EC2)

*Settings → Environments → production → Add secret* (Environment ggf. zuerst anlegen):

| Secret | Wert |
| --- | --- |
| `EC2_HOST` | Public DNS aus Schritt 1 |
| `EC2_USER` | `ec2-user` (Amazon Linux) oder `ubuntu` (Ubuntu-AMI) |
| `EC2_SSH_KEY` | vollständiger Inhalt von `labsuser.pem` (inkl. `BEGIN`/`END`-Zeilen) |

```bash
gh secret set EC2_HOST --env production --body "<public-dns>"
gh secret set EC2_USER --env production --body "<ec2-user|ubuntu>"
gh secret set EC2_SSH_KEY --env production < labsuser.pem
```

### 3b. Repository Secrets (Docker Hub, `docker`-Job)

| Secret | Wert |
| --- | --- |
| `DOCKERHUB_USERNAME` | euer Docker-Hub-Benutzername |
| `DOCKERHUB_TOKEN` | Access Token aus *Docker Hub → Account Settings → Security* (kein Passwort!) |

```bash
gh secret set DOCKERHUB_USERNAME --body "<dockerhub-user>"
gh secret set DOCKERHUB_TOKEN --body "<access-token>"
```

### 3c. Environment `production` Secrets (AWS Academy Learner Lab, `deploy-ecs`-Job)

Werte aus *AWS Details → AWS CLI* im Lab (Schritt 0) — **läuft mit der Session ab,
muss bei jeder neuen Lab-Sitzung wiederholt werden**:

| Secret | Wert |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | `aws_access_key_id` aus dem Lab |
| `AWS_SECRET_ACCESS_KEY` | `aws_secret_access_key` aus dem Lab |
| `AWS_SESSION_TOKEN` | `aws_session_token` aus dem Lab |

Der `deploy-ecs`-Job läuft im Environment `production` — die AWS-Secrets müssen deshalb **im Environment** hinterlegt werden (nicht als Repo-Secrets), damit GitHub sie diesem Job bereitstellt:

```bash
gh secret set AWS_ACCESS_KEY_ID    --env production --body "<...>"
gh secret set AWS_SECRET_ACCESS_KEY --env production --body "<...>"
gh secret set AWS_SESSION_TOKEN     --env production --body "<...>"
```

### 3d. Repository Variables (`build`- und `docker`-Job)

*Settings → Secrets and variables → Actions → Variables*:

| Variable | Wert |
| --- | --- |
| `VITE_API_BASE_URL` | z. B. `http://<ec2-host-des-backends>:3001/` |
| `VITE_IMGS` | `items` (Default, falls nicht gesetzt) |

---

## 4. Kontrolle vor dem Push

- [ ] `gh secret list --env production` zeigt sechs Secrets: `EC2_HOST`, `EC2_USER`, `EC2_SSH_KEY`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`
- [ ] `gh variable list` zeigt `VITE_API_BASE_URL`, `VITE_IMGS`
- [ ] `task-definition.json` enthält `123456789012` als Platzhalter-Account-ID — das ist korrekt, der `deploy-ecs`-Job ersetzt sie zur Laufzeit automatisch
- [ ] ECR-Repo, Cluster, Service, ALB stehen und sind in **us-east-1**

> **Hinweis zum `deploy`-Job:** Der EC2-Deploy-Job hat `continue-on-error: true` gesetzt.
> Falls `EC2_SSH_KEY` nicht konfiguriert ist, schlägt er orange fehl — der Pipeline-Gesamtstatus
> bleibt aber grün und `docker` + `deploy-ecs` laufen durch. Das ist für den reinen ECS-Betrieb
> gewollt: der EC2-Schritt ist optional.

Danach: Push auf `main` (oder `workflow_dispatch`) auslöst alle fünf Jobs;
`deploy`, `docker`, `deploy-ecs` laufen nur bei Push auf `main`, nicht bei Pull Requests.
---

## 5. Neue Lab-Sitzung (nächste Stunde / nächster Tag)

Alle AWS-Ressourcen bleiben zwischen Sitzungen erhalten — EC2-Instanz, VPC, Subnets,
Security Groups, ALB, ECR-Repository, ECS-Cluster und laufende Tasks.
Nur die temporären Zugangsdaten laufen ab — diese drei Schritte genügen:

- [ ] Neues Lab starten, *AWS Details → AWS CLI* öffnen
- [ ] Drei Secrets im Environment `production` aktualisieren:
  ```bash
  gh secret set AWS_ACCESS_KEY_ID    --env production --body "<neuer Wert>"
  gh secret set AWS_SECRET_ACCESS_KEY --env production --body "<neuer Wert>"
  gh secret set AWS_SESSION_TOKEN     --env production --body "<neuer Wert>"
  ```
- [ ] Push auf `main` (oder `workflow_dispatch`) auslösen — alle fünf Jobs laufen durch
