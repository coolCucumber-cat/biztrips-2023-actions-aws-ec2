# EX-03 – Deployment einer Docker-Anwendung auf AWS ECS (Fargate) mit GitHub Actions

## Lernziele

Nach dieser Übung könnt ihr:

- den Unterschied zwischen einem direkten EC2-Deployment (EX-01) und einem containerbasierten ECS-Deployment einordnen
- ein Amazon ECR (Elastic Container Registry) Repository anlegen und ein Image dorthin pushen
- eine ECS-Task-Definition, einen Cluster und einen Service (Fargate-Launch-Type) anlegen
- einen Application Load Balancer vor einen ECS-Service schalten
- eine GitHub-Actions-Pipeline schreiben, die bei jedem Push auf `main` ein neues Image baut, nach ECR pusht und den ECS-Service aktualisiert (Rolling Deployment)
- AWS-Zugangsdaten per OIDC (statt langlebiger Access Keys) an GitHub Actions vergeben

> **Startpunkt:** Diese Übung baut auf dem `docker`-Job aus EX-02 auf, daher vom Tag [`v1-solution`](https://github.com/bbwlc/biztrips-2023-actions-aws-ec2/releases/tag/v1-solution) ausgehen — dort sind `test`, `build`, `deploy` und `docker` bereits vorhanden. Der Tag [`v1-start`](https://github.com/bbwlc/biztrips-2023-actions-aws-ec2/releases/tag/v1-start) reicht hier **nicht**, da ihm auch `deploy` und `docker` fehlen.
>
> Der aktuelle Stand von `main` enthält den `deploy-ecs`-Job bereits fertig (als AWS-Academy-Learner-Lab-Variante, siehe Exkurs nach Schritt 2) — wer die Übung erst selbst durcharbeiten möchte, bevor er sich die Lösung anschaut, arbeitet stattdessen ausgehend von `v1-solution` und ergänzt den Code aus Schritt 7 selbst.

## Voraussetzungen

- Abgeschlossene [EX-02](./EX-02-create-Docker-Image-DockerHub.md) — das Repository enthält bereits ein funktionierendes `Dockerfile`, `.dockerignore` und `nginx.conf`
- Ein AWS-Account mit Rechten, ECR-, ECS-, IAM- und ELB-Ressourcen anzulegen
- AWS CLI lokal installiert und konfiguriert: `aws --version`, `aws sts get-caller-identity`
- `gh` CLI weiterhin eingeloggt (siehe EX-01)

---

## EC2- vs. ECS-Deployment im Vergleich

| | EC2-Deployment (EX-01) | ECS-Deployment (diese Übung) |
| --- | --- | --- |
| **Deploy-Einheit** | Statische Dateien (`dist/`), per `rsync` auf einen konkreten Server kopiert | Ein Docker-Image, das aus einer **Task Definition** heraus als Container gestartet wird |
| **Server-Management** | Ihr verwaltet die EC2-Instanz selbst: OS-Updates, nginx-Installation/-Konfiguration, Prozess am Leben halten | Bei **Fargate** keine Server sichtbar/verwaltbar — AWS betreibt die Rechenkapazität. Bei **EC2-Launch-Type** laufen weiterhin eigene EC2-Instanzen, aber ECS übernimmt das Scheduling der Container darauf |
| **Skalierung** | Manuell, oder selbst eine Auto Scaling Group + Load Balancer aufbauen | Eingebaut über die ECS-**Service**-Definition (`desiredCount`) + Application Auto Scaling |
| **Self-Healing** | Nicht vorhanden — ein abgestürzter Prozess muss manuell/durch ein eigenes Skript neu gestartet werden | Der ECS-Service ersetzt automatisch Tasks, die crashen oder den Health-Check nicht bestehen |
| **Deploy-Mechanismus** | SSH-Verbindung, Dateien kopieren, nginx neu laden | Neues Image nach ECR pushen, Task Definition aktualisieren, `aws ecs update-service` löst ein Rolling Deployment aus — kein SSH nötig |
| **Rollback** | Manuell: alten Build erneut deployen | Alte Task-Definition-Revision erneut als Service-Deployment auswählen |
| **Typische Zugriffskontrolle** | SSH-Key (`EC2_SSH_KEY`) mit vollem Server-Zugriff | IAM-Rolle mit fein granulierten Rechten nur auf ECR/ECS-APIs (idealerweise per OIDC, kein Long-Lived-Key) |
| **Netzwerk** | Direkt gegen die Public-IP/DNS der Instanz, ggf. nginx als Reverse Proxy | Application Load Balancer + Target Group vor dem Service, Health Checks über die ALB |

Kurz gesagt: EX-01 verschiebt Dateien auf einen Server, den ihr komplett selbst betreibt. Diese Übung verschiebt ein **Image** in eine **Registry** und überlässt Scheduling, Skalierung und Self-Healing der Container der ECS-Kontrollebene.

---

## Schritt 1: ECR-Repository anlegen

```bash
aws ecr create-repository --repository-name biztrips --region us-east-1
```

Notiert euch die zurückgegebene `repositoryUri`, z. B. `123456789012.dkr.ecr.us-east-1.amazonaws.com/biztrips` (im Learner Lab immer `us-east-1`).

Überprüfen, ob das Repository angelegt wurde:

```bash
aws ecr describe-repositories --repository-names biztrips --region us-east-1 \
  --query "repositories[0].repositoryUri" --output text
```

---

## Schritt 2: IAM-Rolle für GitHub Actions per OIDC einrichten

Statt langlebiger `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` als Secrets zu hinterlegen (Risiko bei Leak), richtet GitHub als OIDC-Identity-Provider in AWS ein und erstellt eine IAM-Rolle, die GitHub Actions per kurzlebigem Token annehmen kann.

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

Danach eine Rolle `github-actions-biztrips-ecs` mit einer Trust Policy anlegen, die nur auf euer Repository/euren Branch eingeschränkt ist (`repo:<user>/<repo>:ref:refs/heads/main`), und ihr die Policies `AmazonEC2ContainerRegistryPowerUser` sowie eine eingeschränkte ECS-Update-Policy anhängen.

> Details zur Trust-Policy-Syntax: [`aws-actions/configure-aws-credentials`](https://github.com/aws-actions/configure-aws-credentials#configuring-the-role-and-trust-policy) — folgt der dortigen Anleitung, statt die Policy von Hand zu tippen.

---

## Exkurs: Fallback für AWS Academy Learner Lab

Wer diese Übung mit einem **AWS Academy Learner Lab**-Account statt einem
regulären AWS-Account macht, kann Schritt 2 so nicht durchführen: Learner-Lab-
Accounts erlauben kein `iam:CreateOpenIDConnectProvider` und kein
`iam:CreateRole` — es steht nur die vorgegebene `LabRole` zur Verfügung, der
höchstens zusätzliche Policies angehängt werden dürfen. Der Befehl aus
Schritt 2 schlägt dort mit einem `AccessDenied` fehl.

**Fallback:** Statt einer per OIDC angenommenen Rolle die von AWS Academy pro
Lab-Sitzung bereitgestellten temporären Zugangsdaten verwenden. Sie stehen im
Lab unter *AWS Details → AWS CLI* und bestehen aus drei Werten:
`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` und `AWS_SESSION_TOKEN`. Diese
als GitHub Secrets hinterlegen und in Schritt 7 den `configure-aws-
credentials`-Schritt so anpassen:

```yaml
      - name: AWS-Credentials aus Learner-Lab-Session beziehen
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: us-east-1
```

`permissions: id-token: write` wird in diesem Fall nicht gebraucht (kein
OIDC-Token-Request), kann aber im Job stehen bleiben.

Wichtiger Unterschied zum OIDC-Ansatz: Diese Zugangsdaten sind an die
Lab-Sitzung gebunden und laufen ab, sobald die Sitzung endet oder neu
gestartet wird — die GitHub Secrets müssen dann manuell mit den neuen Werten
aktualisiert werden. Das ist eine Einschränkung der Lab-Sandbox, kein
empfohlenes Produktions-Pattern: In einem regulären AWS-Account bleibt der
OIDC-Ansatz aus Schritt 2 der richtige Weg, weil er ganz ohne gespeicherte
Zugangsdaten auskommt und nicht manuell erneuert werden muss.

**Auswirkung auf die `executionRoleArn` in Schritt 5:** Die `LabRole`-
Einschränkung gilt nicht nur für die per OIDC angenommene Rolle aus Schritt 2,
sondern für **jede** eigene IAM-Rolle — auch für die in Schritt 5 verwendete
`ecsTaskExecutionRole`. `iam:CreateRole` schlägt im Learner Lab ebenso fehl,
wenn versucht wird, diese Rolle separat anzulegen. Im Learner Lab in der
`task-definition.json` daher direkt die vorgegebene `LabRole` referenzieren:

```json
"executionRoleArn": "arn:aws:iam::<account-id>:role/LabRole",
```

Die AWS-Region der Learner-Lab-Sitzung ist ebenfalls fix `us-east-1` (siehe
Region im Fallback-Snippet oben) — das betrifft dann auch die
`awslogs-region` sowie die ECR-Image-URI in der `task-definition.json` und
den Cluster/Service aus Schritt 3/6, die entsprechend in `us-east-1` statt
`eu-central-1` angelegt werden müssen.

---

## Schritt 3: ECS-Cluster anlegen

```bash
aws ecs create-cluster --cluster-name biztrips-cluster --region us-east-1
```

Ein Fargate-Cluster braucht keine eigenen EC2-Instanzen — der Cluster ist zunächst nur ein logischer Namespace für Services/Tasks. Prüfen, ob der Cluster aktiv ist:

```bash
aws ecs describe-clusters --clusters biztrips-cluster --region us-east-1 \
  --query "clusters[0].status" --output text
# Erwartete Ausgabe: ACTIVE
```

---

## Schritt 4: Application Load Balancer + Target Group

Fargate-Tasks bekommen eine ENI mit eigener IP (kein fester EC2-Host), weshalb die Target Group den Typ `ip` braucht. Der ALB sitzt davor und prüft die Gesundheit der Tasks über den Health-Check-Pfad `/`.

### 4a. VPC und Subnets ermitteln

Im Learner Lab gibt es eine Default-VPC. Deren ID und die zugehörigen Subnets (mindestens zwei für den ALB) nachschlagen:

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" --output text --region us-east-1)
echo "VPC: $VPC_ID"

# Alle Subnet-IDs der Default-VPC (durch Leerzeichen getrennt)
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[*].SubnetId" --output text --region us-east-1
```

Zwei der zurückgegebenen Subnet-IDs notieren (z. B. `subnet-aaa111` und `subnet-bbb222`).

### 4b. Security Groups anlegen

```bash
# Security Group für den ALB (eingehend Port 80 aus dem Internet)
SG_ALB=$(aws ec2 create-security-group \
  --group-name biztrips-alb-sg \
  --description "ALB biztrips" \
  --vpc-id $VPC_ID \
  --query GroupId --output text --region us-east-1)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ALB --protocol tcp --port 80 --cidr 0.0.0.0/0 \
  --region us-east-1

echo "ALB-SG: $SG_ALB"

# Security Group für die Fargate-Tasks (Port 80 NUR von der ALB-SG)
SG_TASKS=$(aws ec2 create-security-group \
  --group-name biztrips-tasks-sg \
  --description "ECS Tasks biztrips" \
  --vpc-id $VPC_ID \
  --query GroupId --output text --region us-east-1)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_TASKS --protocol tcp --port 80 \
  --source-group $SG_ALB --region us-east-1

echo "Tasks-SG: $SG_TASKS"
```

> **Wichtig:** Die Task-SG darf Port 80 **ausschliesslich** von `$SG_ALB` erlauben. Direktzugriff aus dem Internet soll nur über den ALB erfolgen — dieser Aufbau ist der Punkt, an dem Security-Group-Fehler später zu `HealthCheck failed`-Fehlern führen.

### 4c. Target Group anlegen

```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name biztrips-tg \
  --protocol HTTP --port 80 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-path / \
  --query "TargetGroups[0].TargetGroupArn" --output text --region us-east-1)

echo "Target Group ARN: $TG_ARN"
```

### 4d. Application Load Balancer + Listener anlegen

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name biztrips-alb \
  --subnets subnet-aaa111 subnet-bbb222 \
  --security-groups $SG_ALB \
  --query "LoadBalancers[0].LoadBalancerArn" --output text --region us-east-1)

echo "ALB ARN: $ALB_ARN"

# Listener Port 80 → Target Group
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN \
  --region us-east-1

# ALB-DNS-Namen notieren (für spätere Tests)
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query "LoadBalancers[0].DNSName" --output text --region us-east-1)

echo "ALB DNS: $ALB_DNS"
```

`http://$ALB_DNS` wird nach erfolgreichem ECS-Deploy die App ausliefern.

---

## Schritt 5: Task Definition schreiben

`task-definition.json` im Projektroot:

```json
{
  "family": "biztrips",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/LabRole",
  "containerDefinitions": [
    {
      "name": "biztrips",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/biztrips:latest",
      "portMappings": [{ "containerPort": 80, "protocol": "tcp" }],
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/biztrips",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs",
          "awslogs-create-group": "true"
        }
      }
    }
  ]
}
```

`executionRoleArn` zeigt auf die `LabRole` des Learner Labs — diese vorgegebene Rolle hat die Policy `AmazonECSTaskExecutionRolePolicy` angehängt und erlaubt dem Container, das Image von ECR zu ziehen und Logs nach CloudWatch zu schreiben (analog zur Rolle, die in EX-01 der SSH-User implizit über `sudo`-Rechte auf der Instanz hatte).

> **`awslogs-create-group: "true"`** — ohne dieses Flag schlägt der Task-Start fehl, wenn die CloudWatch-Log-Gruppe `/ecs/biztrips` noch nicht existiert. Mit dem Flag legt der Container-Agent sie beim ersten Start automatisch an, ohne dass vorher `aws logs create-log-group` ausgeführt werden muss.

> **Regulärer AWS-Account:** Wer nicht mit dem Learner Lab arbeitet, legt eine eigene Rolle `ecsTaskExecutionRole` mit der Policy `AmazonECSTaskExecutionRolePolicy` an und trägt deren ARN ein. Im Learner Lab ist `iam:CreateRole` gesperrt — dort bleibt es bei `LabRole`.

---

## Exkurs: Muss das Image in ECR liegen?

Nein. ECS/Fargate kann Images grundsätzlich aus jeder Registry ziehen, nicht nur aus ECR — auch aus DockerHub. Diese Übung verwendet ECR, weil die `executionRoleArn` den Pull dann ohne zusätzliche Zugangsdaten erlaubt (rein über IAM) und kein Internetzugriff der Tasks nötig ist. Bei DockerHub sieht es je nach Sichtbarkeit des Images anders aus:

**Öffentliches DockerHub-Image**
Einfach die Image-URL direkt in der Task Definition eintragen, z. B. `"image": "docker.io/<user>/biztrips:latest"` — kein ECR-Push-Schritt nötig. Wichtig: Die Fargate-Tasks brauchen dann **Internetzugriff** (Public IP in einem öffentlichen Subnet oder NAT-Gateway), da DockerHub im Gegensatz zu ECR nicht über einen AWS-internen Pfad erreichbar ist. Fehlt das, schlägt der Pull mit `CannotPullContainerError` fehl (siehe Stolpersteine).

**Privates DockerHub-Repo**
Zusätzlich müssen Zugangsdaten hinterlegt werden:

1. DockerHub-Zugangsdaten als Secret in AWS Secrets Manager anlegen, z. B.:
   ```bash
   aws secretsmanager create-secret \
     --name dockerhub-credentials \
     --secret-string '{"username":"<user>","password":"<token>"}'
   ```
2. In der Task Definition beim Container `repositoryCredentials` referenzieren:
   ```json
   {
     "repositoryCredentials": {
       "credentialsParameter": "arn:aws:secretsmanager:eu-central-1:123456789012:secret:dockerhub-credentials"
     }
   }
   ```
3. Die `executionRoleArn` braucht zusätzlich `secretsmanager:GetSecretValue` auf dieses Secret, sonst schlägt der Pull mit einem Berechtigungsfehler fehl.

Kurz: ECR ist hier die einfachere, tiefer integrierte Lösung ohne separates Credential-Handling — DockerHub funktioniert aber genauso, mit etwas mehr Konfigurationsaufwand.

---

## Schritt 6: ECS-Service anlegen

Bevor der Service angelegt wird, müssen die IDs aus Schritt 4 bekannt sein. Falls die Shell-Variablen nicht mehr gesetzt sind, können sie so nachgeschlagen werden:

```bash
# Subnet-IDs der Default-VPC
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" --output text --region us-east-1)

SUBNET_1=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" --output text --region us-east-1)

SUBNET_2=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[1].SubnetId" --output text --region us-east-1)

# Security Group der Tasks
SG_TASKS=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=biztrips-tasks-sg" \
  --query "SecurityGroups[0].GroupId" --output text --region us-east-1)

# Target Group ARN
TG_ARN=$(aws elbv2 describe-target-groups --names biztrips-tg \
  --query "TargetGroups[0].TargetGroupArn" --output text --region us-east-1)

echo "Subnets: $SUBNET_1 $SUBNET_2"
echo "Tasks-SG: $SG_TASKS"
echo "TG ARN: $TG_ARN"
```

Task Definition registrieren und den ECS-Service anlegen:

```bash
# Task Definition aus task-definition.json registrieren
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json --region us-east-1

# ECS-Service anlegen
aws ecs create-service \
  --cluster biztrips-cluster \
  --service-name biztrips-service \
  --task-definition biztrips \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_1,$SUBNET_2],securityGroups=[$SG_TASKS],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=biztrips,containerPort=80" \
  --region us-east-1
```

> **`assignPublicIp=ENABLED`** ist für Tasks in einem öffentlichen Subnet ohne NAT-Gateway zwingend — ohne Public IP kann der Task-Agent weder das Image aus ECR ziehen noch Logs an CloudWatch senden.

Prüfen, ob die Tasks hochkommen:

```bash
aws ecs list-tasks --cluster biztrips-cluster --service-name biztrips-service \
  --region us-east-1

# Detailstatus eines Tasks (PROVISIONING → PENDING → RUNNING = alles ok)
aws ecs describe-tasks --cluster biztrips-cluster \
  --tasks <task-arn-aus-list-tasks> --region us-east-1 \
  --query "tasks[0].lastStatus" --output text
```

`desired-count 2` sorgt dafür, dass immer zwei Tasks laufen — fällt eine aus, ersetzt der Service sie automatisch. Das ist der Punkt, an dem ECS sich am deutlichsten von EX-01 unterscheidet: Dort gab es genau **eine** Instanz ohne eingebaute Redundanz.

---

## Schritt 7: GitHub-Actions-Job `deploy-ecs`

Neuer Job in `.github/workflows/deploy.yml`, der auf den bestehenden `docker`-Job aus EX-02 aufbaut. Das Beispiel unten zeigt die im Repo enthaltene **Learner-Lab-Variante** (temporäre Zugangsdaten statt OIDC):

```yaml
  deploy-ecs:
    name: Image nach ECR pushen und ECS-Service aktualisieren
    runs-on: ubuntu-latest
    needs: docker
    if: github.ref == 'refs/heads/main' && github.event_name != 'pull_request'
    environment: production
    steps:
      - uses: actions/checkout@v4

      # AWS-Academy-Learner-Lab-Fallback: Lab-Accounts erlauben kein iam:CreateRole /
      # iam:CreateOpenIDConnectProvider, daher hier die temporären Learner-Lab-
      # Zugangsdaten (AWS Details → AWS CLI im Lab) statt einer per OIDC übernommenen
      # Rolle. Diese Zugangsdaten laufen mit der Lab-Sitzung ab und müssen bei jeder
      # neuen Sitzung als GitHub Secrets im "production"-Environment aktualisiert werden.
      - name: AWS-Credentials aus Learner-Lab-Session beziehen
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: us-east-1

      - name: Bei ECR anmelden
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      # task-definition.json enthält einen Platzhalter-Account (123456789012) für
      # executionRoleArn, da die Learner-Lab-Account-ID sich pro Sitzung/Kurs
      # unterscheidet. Hier wird sie live über die aktuellen Zugangsdaten ermittelt
      # und in der Task-Definition ersetzt, statt sie manuell pflegen zu müssen.
      - name: Account-ID der Learner-Lab-Session ermitteln
        id: aws-account
        run: echo "id=$(aws sts get-caller-identity --query Account --output text)" >> "$GITHUB_OUTPUT"

      - name: executionRoleArn in Task-Definition aktualisieren
        run: |
          sed -i "s#arn:aws:iam::[0-9]*:role/LabRole#arn:aws:iam::${{ steps.aws-account.outputs.id }}:role/LabRole#" task-definition.json

      - name: Image bauen und nach ECR pushen
        env:
          ECR_REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          ECR_REPOSITORY: biztrips
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build \
            --build-arg VITE_API_BASE_URL="${{ vars.VITE_API_BASE_URL }}" \
            --build-arg VITE_IMGS="${{ vars.VITE_IMGS || 'items' }}" \
            -t "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" \
            -t "$ECR_REGISTRY/$ECR_REPOSITORY:latest" .
          docker push "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"
          docker push "$ECR_REGISTRY/$ECR_REPOSITORY:latest"

      - name: Task-Definition mit neuem Image aktualisieren
        id: render-task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: biztrips
          image: ${{ steps.ecr-login.outputs.registry }}/biztrips:${{ github.sha }}

      - name: ECS-Service deployen
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: ${{ steps.render-task-def.outputs.task-definition }}
          cluster: biztrips-cluster
          service: biztrips-service
          wait-for-service-stability: true
```

Wichtige Design-Entscheidungen:

- **Account-ID per `aws sts get-caller-identity`** — die Learner-Lab-Account-ID wechselt mit jeder Sitzung. Statt sie hart zu codieren, wird sie live aus den temporären Zugangsdaten gelesen und per `sed` in `task-definition.json` eingetragen. Damit muss die Datei nie manuell angefasst werden.
- **`wait-for-service-stability: true`** — der Job wartet, bis ECS bestätigt, dass alle neuen Tasks laufen und die alten abgelöst wurden (Rolling Deployment), statt sofort grün zu melden, während im Hintergrund noch deployt wird.
- Kein SSH-Key, kein `rsync` — der einzige Credentials-Bedarf sind die temporären Lab-Zugangsdaten aus den GitHub Secrets `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`.
- **`environment: production`** — bindet den Job an das Production-Environment in GitHub Settings, in dem die AWS-Secrets hinterlegt sind.

> **Regulärer AWS-Account (OIDC):** Wer kein Learner Lab verwendet, ersetzt den `configure-aws-credentials`-Schritt durch die OIDC-Variante (`role-to-assume: <arn>`) aus dem Exkurs nach Schritt 2 und fügt `permissions: id-token: write` zum Job hinzu. Die Account-ID-Ermittlung und der `sed`-Schritt entfallen dann.

### Benötigte GitHub Secrets (`production` Environment)

| Secret | Wert | Wann aktualisieren |
| --- | --- | --- |
| `AWS_ACCESS_KEY_ID` | aus *AWS Details → AWS CLI* im Lab | bei jeder neuen Lab-Sitzung |
| `AWS_SECRET_ACCESS_KEY` | aus *AWS Details → AWS CLI* im Lab | bei jeder neuen Lab-Sitzung |
| `AWS_SESSION_TOKEN` | aus *AWS Details → AWS CLI* im Lab | bei jeder neuen Lab-Sitzung |

### Exkurs: Repository Secrets vs. Environment Secrets

In GitHub Actions gibt es zwei Orte, an denen Secrets hinterlegt werden können — und die Wahl entscheidet, welche Jobs sie lesen dürfen:

| | Repository Secret | Environment Secret |
| --- | --- | --- |
| **Hinterlegt unter** | *Settings → Secrets and variables → Actions → Secrets* | *Settings → Environments → `<name>` → Secrets* |
| **Sichtbar für** | **alle** Jobs in allen Workflows des Repos | nur Jobs, die `environment: <name>` deklarieren |
| **Schutzregeln** | keine | optional: Pflicht-Reviewer, Branch-Einschränkung, Wartezeit |
| **`gh`-Befehl** | `gh secret set NAME --body "..."` | `gh secret set NAME --env production --body "..."` |

**Warum das in diesem Repo so aufgeteilt ist:**

- `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` → **Repository Secret** — der `docker`-Job hat kein `environment:` gesetzt und kann deshalb nur Repo-Secrets lesen. Docker Hub ist kein Deployment-Ziel mit Schutzanforderungen.
- `EC2_HOST`, `EC2_USER`, `EC2_SSH_KEY` → **Environment Secret `production`** — nur der `deploy`-Job (mit `environment: production`) darf sie lesen. Ein Angreifer, der einen PR öffnet, kann diese Secrets nicht aus einem PR-Build extrahieren.
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` → **Environment Secret `production`** — aus demselben Grund: der `deploy-ecs`-Job deklariert `environment: production`, alle anderen Jobs bleiben blind.

Die Faustregel: **Zugangsdaten für Produktions-Deployments gehören ins Environment**, nicht ins Repo. Damit lassen sich später Schutzregeln (z. B. manueller Approve-Schritt vor jedem Prod-Deploy) ergänzen, ohne den Workflow umzuschreiben.

---

## Bekannte Stolpersteine

**`AccessDenied` beim `configure-aws-credentials`-Schritt**
Die Trust Policy der IAM-Rolle ist meist zu eng oder zu weit falsch konfiguriert — prüft, ob `sub` in der Trust Policy exakt `repo:<user>/<repo>:ref:refs/heads/main` entspricht (inkl. korrektem Repo-Namen und Branch).

**`AccessDenied` schon bei `aws iam create-open-id-connect-provider` (Schritt 2)**
Typisch für **AWS Academy Learner Lab**-Accounts — dort ist das Anlegen eigener IAM-Rollen/OIDC-Provider grundsätzlich gesperrt. Siehe den Exkurs oben zum Fallback mit den temporären Learner-Lab-Zugangsdaten.

**Task startet, aber Health-Check der Target Group schlägt dauerhaft fehl**
Meist Security-Group-Problem: Die Security Group der Tasks muss eingehenden Traffic von der Security Group des ALB auf Port 80 erlauben (Schritt 4) — nicht umgekehrt.

**`CannotPullContainerError` beim Task-Start**
Die `executionRoleArn` fehlt oder hat nicht die Policy `AmazonECSTaskExecutionRolePolicy` — ohne diese Rolle darf der Task-Agent das Image nicht von ECR ziehen. Im **AWS Academy Learner Lab** genügt es nicht, eine eigene `ecsTaskExecutionRole` anzulegen (schlägt mit `AccessDenied` fehl) — dort `executionRoleArn` auf die vorgegebene `LabRole` setzen (siehe Exkurs nach Schritt 2).

**Service bleibt bei "PENDING", `desired-count` wird nie erreicht**
Häufig fehlende `assignPublicIp=ENABLED` bei Tasks in einem öffentlichen Subnet ohne NAT-Gateway — ohne Public IP kann der Task-Agent weder das Image ziehen noch Logs senden.

---

## Reflexionsfragen

1. Welche Aufgaben, die in EX-01 der SSH-Deploy-Schritt (`rsync` + `sudo systemctl reload nginx`) manuell erledigt hat, übernimmt in dieser Übung die ECS-Kontrollebene automatisch?
2. Warum ist eine über OIDC bezogene, kurzlebige IAM-Rolle sicherer als ein dauerhaft in GitHub Secrets hinterlegter AWS-Access-Key, wie ihn EX-01 in Form von `EC2_SSH_KEY` verwendet?
3. Was passiert mit den zwei laufenden Tasks eines Services, wenn ein `update-service` mit neuer Task-Definition ausgelöst wird — und warum ist das ein Vorteil gegenüber dem Single-Server-Deployment aus EX-01?
4. Warum braucht die Target Group in Schritt 4 den Typ `ip` statt `instance`, obwohl EX-01/EX-02 nie mit Target Groups gearbeitet haben?
