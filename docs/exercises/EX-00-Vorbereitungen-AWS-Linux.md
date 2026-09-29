# EX-00 — Preparing AWS-Linux-How to deploy on an AWS Academy EC2 instance

> Possible solution for the exercise "Deploy the biztrips build to an EC2 instance"
> in the AWS Academy **Learner Lab**, driven by the existing GitLab CI/CD pipeline.
>
> Steps 2, 3 and the deploy commands of step 5 were run against a live Learner Lab
> instance (Amazon Linux 2023, nginx 1.30) on 2026-09-01: 19 files published,
> `/` → 200, hashed assets served with the correct content type, SPA fallback working.

## Goal

The pipeline has three stages (`test` → `build` → `deploy`) and already publishes the
CRA build (`build/`) to Netlify. In this exercise we add a **second deploy target**
without removing the first one: an nginx web server on an EC2 instance in the
AWS Academy Learner Lab.

After the exercise, one push to `main` gives you:

| Target | Job | Trigger | Result |
|---|---|---|---|
| AWS EC2 | `deploy ec2` | **automatic** on `main` | `http://<public-dns>/` |

Both jobs sit in the same `deploy` stage and consume the same build artifact — that
is the didactic point: *one build, several deployment targets*. The EC2 job is manual
because the Learner Lab instance is stopped outside a lab session, and an automatic
job would turn every pipeline red.

The follow-up exercises deploy the same application with **GitHub Actions**. Use a
**second, separate EC2 instance** for those (see [Variant B](#variant-b--one-shared-instance-for-both-pipelines)
if you prefer to share a single instance).

---

## 0. What is special about AWS Academy

The Learner Lab is not a normal AWS account. These restrictions drive most of the
design decisions below:

| Restriction | Consequence for us |
|---|---|
| A lab session lasts max. 4 hours; when it ends, running EC2 instances are **stopped** (not terminated) | The deploy job must not run automatically — it would fail while the lab is off. We use `when: manual`. |
| The public IPv4 / public DNS changes on every stop → start | The host is a **CI/CD variable**, never hard-coded in `.gitlab-ci.yml`. Update it at the start of each lab session. |
| You cannot create IAM users, roles or access keys | We deploy over **SSH with a key pair**, not with the AWS CLI / S3 / CodeDeploy. |
| A key pair named `vockey` is pre-created; the private key `labsuser.pem` is downloadable from "AWS Details" | That is the key we hand to the GitLab runner. |
| Only small instance types (t2/t3.micro … medium) and a limited credit budget | `t3.micro` is plenty for a static site. **Stop the instance when you are done.** |

---

## 1. Start the lab and launch the instance
c
1. In AWS Academy: **Start Lab**, wait until the dot next to "AWS" is green, then open the AWS console.
2. EC2 → **Launch instance**
    - Name: `biztrips-frontend`
    - AMI: **Amazon Linux 2023** (login user is `ec2-user`; on an Ubuntu AMI it is `ubuntu`)
    - Instance type: `t3.micro`
    - Key pair: **vockey**
    - Network settings → *Create security group* with two inbound rules:
        - `SSH` (22) — source `0.0.0.0/0` (see the note in step 4)
        - `HTTP` (80) — source `0.0.0.0/0`
3. Launch, then note the **Public IPv4 DNS** (e.g. `ec2-3-91-12-34.compute-1.amazonaws.com`).

> If your lab allows allocating an **Elastic IP**, associate one now — it survives
> stop/start and saves you from updating the CI variable every session.

### 1b. Verify the instance is actually reachable — before touching anything else

Do not skip this. A pipeline that cannot reach the instance fails in a way that looks
like a *pipeline* problem, and you will debug the wrong layer for an hour.

**First, check the rules where they count.** The launch wizard often creates a group
called `launch-wizard-N`, while the console's *Security Groups* page conveniently opens
`default` — so it is easy to add perfect rules to a group that is not attached to your
instance. The one view that cannot lie:

> EC2 → **Instances** → select the instance → **Security** tab

It shows the attached security group **and** its inbound rules together. If your SSH and
HTTP rules are not listed *there*, you edited the wrong group. While you are on that
screen, confirm **Status checks** reads *2/2 passed*.

**Then test from your own machine**, before any CI is involved:

```bash
nc -z -v <public-dns> 22     # expect: succeeded!
nc -z -v <public-dns> 80     # expect: succeeded! (once nginx runs, step 2)
ssh -i labsuser.pem ec2-user@<public-dns> true && echo "ssh ok"
```

**Read the failure mode — it tells you which layer is broken:**

| What you get | What it means |
|---|---|
| `succeeded!` | The port is open and something is listening. |
| **Operation timed out** | Packets are being **dropped** — nothing ever reaches the instance. This is a **security group** (or subnet routing) problem. The host is not involved. |
| **Connection refused** | Packets *arrive*, but no process is listening on that port. The network is fine; the **service** is not running (e.g. nginx not started yet). |

That distinction is the single most useful thing in this document. A timeout is never
fixed by restarting nginx, and a refusal is never fixed by editing a security group.

## 2. Prepare the web server (once per instance)

Download `labsuser.pem` from **AWS Details → Download PEM**, then:

```bash
chmod 400 labsuser.pem
ssh -i labsuser.pem ec2-user@<public-dns>
```

On the instance:

```bash
sudo dnf install -y nginx rsync
sudo systemctl enable --now nginx

# document root for our app
sudo mkdir -p /var/www/biztrips
sudo chown -R ec2-user:ec2-user /var/www/biztrips
```

Point nginx at that directory — `sudo vi /etc/nginx/conf.d/biztrips.conf`:

```nginx
server {
    listen 80 default_server;
    server_name _;
    root /var/www/biztrips;
    index index.html;

    # single-page app: unknown paths fall back to index.html
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

> **Watch the `try_files` line.** It must be `try_files $uri $uri/ /index.html;`.
> Leaving out the first `$uri` — easy to do when retyping this in `vi` — produces a
> site that *looks* fine: `/` returns 200. But no real file is ever served directly
> any more, so every JS and CSS request also falls back to `index.html`, the browser
> receives HTML where it expects JavaScript, and the page renders blank. Verify with
> `curl -I http://<public-dns>/static/js/main.<hash>.js` — the `Content-Type` must be
> `application/javascript`, not `text/html`.

Amazon Linux ships a stock server block in `/etc/nginx/nginx.conf` that also listens
on 80. It has no `default_server`, so nginx still starts — you just get a
`conflicting server name "_" on 0.0.0.0:80, ignored` warning on every reload. Move it
out of the way (verified on Amazon Linux 2023, nginx 1.30):

```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
sudo sed -i 's/^        listen       80;/        listen       8081;/; s/^        listen       \[::\]:80;/        listen       [::]:8081;/' /etc/nginx/nginx.conf
```

Then:

```bash
sudo nginx -t && sudo systemctl reload nginx
echo "hello from ec2" > /var/www/biztrips/index.html
```

Open `http://<public-dns>/` in the browser — you should see `hello from ec2`.
If not, fix that **before** touching the pipeline: the problem is the security
group or nginx, not GitLab.

## 3. Give the runner a place to write

The CI job copies into a staging directory owned by the login user and then
syncs it into the document root with `sudo`. Passwordless sudo must therefore
work — on Amazon Linux, `ec2-user` already has full `NOPASSWD` sudo, so there is
nothing to do. On a hardened image add:

```bash
echo 'ec2-user ALL=(ALL) NOPASSWD: /usr/bin/rsync, /usr/bin/systemctl' | sudo tee /etc/sudoers.d/deploy
```
