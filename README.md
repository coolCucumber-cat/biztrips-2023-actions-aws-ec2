# RefCard 02 – biztrips

React 19 + Vite 8 Anwendung mit GitHub-Actions-Pipeline und Deployment auf AWS EC2.

Teil der RefCard-Sequenz für das Modul M324 DevOps (CI/CD-Einheit): RefCard 02
deckt Frontend-CI/CD (GitHub Actions, EC2, Docker, ECS) ab. Die geplante
Fortsetzung (Spring-Boot-Backend, Storage, Fullstack auf AWS) ist in
[`docs/future/RefCard-03-RefCard-05-own-repos.md`](docs/future/RefCard-03-RefCard-05-own-repos.md)
festgehalten.

## Tags für Studierende

| Tag | Inhalt |
| --- | --- |
| [`v1-start`](https://github.com/bbwlc/biztrips-2023-actions-aws-ec2/releases/tag/v1-start) | Startpunkt für die Übungen: `deploy.yml` enthält nur die Jobs `test` und `build`. Die Jobs `deploy` (EX-01) und `docker` (EX-02) fehlen und sind selbst zu ergänzen. |
| [`v1-solution`](https://github.com/bbwlc/biztrips-2023-actions-aws-ec2/releases/tag/v1-solution) | Referenzlösung (Stand `main`): vollständige Pipeline mit `test`, `build`, `deploy` (EC2) und `docker` (Docker Hub). |

Mit dem Startpunkt arbeiten:

```bash
git checkout v1-start
```

Bei Bedarf zum Vergleich die Lösung ansehen:

```bash
git diff v1-start v1-solution -- .github/workflows/deploy.yml
```

> [EX-03](docs/exercises/EX-03-deploy-AWS-ECS.md) (ECS/Fargate): Der
> `deploy-ecs`-Job ist seit [PR #1](https://github.com/bbwlc/biztrips-2023-actions-aws-ec2/pull/1)
> auf `main` als Referenzlösung vorhanden, in der AWS-Academy-Learner-Lab-
> Variante (temporäre Zugangsdaten statt OIDC, Region `us-east-1`, `LabRole`
> statt eigener `ecsTaskExecutionRole` — siehe Exkurs in EX-03). Wer die
> Übung ohne vorherigen Blick auf `main` durchführen möchte, arbeitet
> stattdessen mit `v1-solution` und ergänzt den Job selbst gemäss EX-03
> Schritt 7.

## Voraussetzungen

- Node.js 24 oder neuer
- npm

## Lokale Entwicklung

```bash
npm install
npm run dev
```

`npm run dev` startet den Vite-Dev-Server auf http://localhost:3000.

## Verfügbare Scripts

| Script | Beschreibung |
| --- | --- |
| `npm run dev` | Den Vite-Dev-Server starten |
| `npm run build` | Produktions-Build nach `dist/` erzeugen |
| `npm run preview` | Produktions-Build lokal ausliefern |
| `npm test` | Tests einmalig mit Vitest ausführen |
| `npm run test:watch` | Tests im Watch-Modus ausführen |

## Umgebungsvariablen

Vite liest nur Variablen mit dem Präfix `VITE_`. Die lokalen Standardwerte
stehen in `.env`:

| Variable | Bedeutung |
| --- | --- |
| `VITE_API_BASE_URL` | Basis-URL der Backend-API |
| `VITE_IMGS` | Pfadsegment für Bilder |

Im Code werden sie über `import.meta.env.VITE_API_BASE_URL` gelesen.

## CI/CD-Pipeline

Die Pipeline liegt in `.github/workflows/deploy.yml` und besteht aus fünf Jobs:

1. **test** – `npm ci` und `npm test` (Vitest)
2. **build** – `npm run build`, das Ergebnis aus `dist/` wird als Artefakt hochgeladen
3. **deploy** – lädt das Artefakt, überträgt es per `rsync` über SSH auf die
   EC2-Instanz nach `/var/www/biztrips` und lädt nginx neu
4. **docker** – baut das Docker-Image und veröffentlicht es auf Docker Hub
5. **deploy-ecs** – pusht das Image zusätzlich nach ECR und aktualisiert einen
   ECS-Service (Fargate) per Rolling Deployment (siehe [EX-03](docs/exercises/EX-03-deploy-AWS-ECS.md))

`deploy`, `docker` und `deploy-ecs` laufen nur bei Pushes auf `main`, nicht bei Pull Requests.

### Benötigte GitHub Secrets

Unter *Settings → Secrets and variables → Actions → Secrets*:

| Secret | Beispiel | Beschreibung |
| --- | --- | --- |
| `EC2_HOST` | `ec2-1-2-3-4.eu-central-1.compute.amazonaws.com` | Hostname oder IP der EC2-Instanz |
| `EC2_USER` | `ubuntu` | SSH-Benutzer (`ubuntu` bei Ubuntu-AMI, `ec2-user` bei Amazon Linux) |
| `EC2_SSH_KEY` | `-----BEGIN OPENSSH PRIVATE KEY-----…` | Privater SSH-Key, vollständig inklusive Kopf- und Fusszeile |
| `DOCKERHUB_USERNAME` | `<dockerhub-user>` | Docker-Hub-Benutzername für den `docker`-Job |
| `DOCKERHUB_TOKEN` | `dckr_pat_…` | Docker-Hub-Access-Token für den `docker`-Job |
| `AWS_ACCESS_KEY_ID` | `ASIA…` | AWS-Academy-Learner-Lab-Zugangsdaten für `deploy-ecs` (aus *AWS Details → AWS CLI* im Lab, läuft mit der Lab-Sitzung ab) |
| `AWS_SECRET_ACCESS_KEY` | `…` | s.o. |
| `AWS_SESSION_TOKEN` | `…` | s.o. |

### Benötigte GitHub Variables

Unter *Settings → Secrets and variables → Actions → Variables*:

| Variable | Beispiel |
| --- | --- |
| `VITE_API_BASE_URL` | `http://ec2-1-2-3-4.eu-central-1.compute.amazonaws.com:3001/` |
| `VITE_IMGS` | `items` |

### Environment

Der Deploy-Job nutzt das Environment `production`. Es muss unter
*Settings → Environments* angelegt werden, sonst schlägt der Job fehl.

## Setup der EC2-Instanz

Einmalig auf der Instanz ausführen:

```bash
sudo apt update
sudo apt install -y nginx rsync
sudo mkdir -p /var/www/biztrips
sudo chown -R "$USER":"$USER" /var/www/biztrips
```

nginx-Konfiguration unter `/etc/nginx/sites-available/biztrips`:

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/biztrips;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Aktivieren und neu laden:

```bash
sudo ln -s /etc/nginx/sites-available/biztrips /etc/nginx/sites-enabled/biztrips
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Der Deploy-Job ruft `sudo` ohne Passwort auf. Dafür muss der SSH-Benutzer
passwortloses sudo besitzen (bei den AWS-Standard-AMIs für `ubuntu` bzw.
`ec2-user` bereits der Fall). In der Security Group müssen Port 22 für den
GitHub-Runner und Port 80 für die Besucher offen sein.