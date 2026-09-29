# AWS CLI Profile verwenden

`aws sts get-caller-identity` lädt standardmässig das **default**-Profil. Um mit mehreren AWS-Accounts/Usern zu arbeiten, legt man zusätzliche Profile an und wählt sie explizit aus.

## Neues Profil anlegen

```bash
aws configure --profile mein-profil
```

Fragt nach `AWS Access Key ID`, `AWS Secret Access Key`, `Default region name`, `Default output format` und speichert sie unter dem angegebenen Profilnamen.

## Profile anzeigen

```bash
aws configure list-profiles
```

## Profil für einen einzelnen Befehl verwenden

```bash
aws sts get-caller-identity --profile mein-profil
```

## Profil für die aktuelle Shell-Session setzen

```bash
export AWS_PROFILE=mein-profil
aws sts get-caller-identity
```

Gilt nur für die aktuelle Terminal-Session. Zurück zum default-Profil:

```bash
unset AWS_PROFILE
```

## Konfigurationsdateien

Profile werden in zwei Dateien gespeichert:

- `~/.aws/credentials` – Access Key / Secret Key
- `~/.aws/config` – Region, Output-Format, ggf. Rollen

Beispiel `~/.aws/credentials`:

```ini
[default]
aws_access_key_id = ...
aws_secret_access_key = ...

[mein-profil]
aws_access_key_id = ...
aws_secret_access_key = ...
```
