# Local Family Cloud — MVP 1

This repository runs a local family cloud on one Windows 11 computer:

- Nextcloud stores documents and general files.
- Immich stores photos and videos and receives automatic phone backups.
- Caddy provides the local `drive.tolba` and `photos.tolba` HTTP addresses.
- Docker Compose manages both applications and their private databases/caches.

The system is for a trusted home LAN only. It deliberately has no HTTPS,
internet exposure, port forwarding, VPN, custom application, document RAG, or
advanced AI search.

## Architecture

| Application | Containers | Host port | Persistent storage |
| --- | --- | --- | --- |
| Nextcloud | Apache, cron, MariaDB, Redis | `45321` | `family-cloud-nextcloud-html`, `family-cloud-nextcloud-data`, `family-cloud-nextcloud-db` |
| Immich | Server, PostgreSQL, Valkey | `12345` | `storage/immich/library`, `family-cloud-immich-db` |
| Local proxy | Caddy | `80` | None; configuration is tracked in `Caddyfile` |

MariaDB, PostgreSQL, Redis, and Valkey have no host ports. The Nextcloud and
Immich networks are separate. Redis and Valkey are disposable caches; they do
not need backups.

Immich machine learning is intentionally not included in MVP1. After the first
Immich login, disable its Smart Search and Facial Recognition jobs as described
below so the server does not queue work for a machine-learning container.

## Storage layout

```text
drive/
├── storage/
│   └── immich/library/       # Immich originals, thumbnails and DB dumps
├── backups/
│   ├── nextcloud/database/   # Temporary Nextcloud SQL dump location
│   └── immich/database/      # Temporary Immich SQL dump location
├── .env                      # Local secrets; ignored by Git
├── .env.example
└── docker-compose.yml
```

The databases and all Nextcloud files use named Docker volumes inside Docker
Desktop's Linux VM. This avoids the permission problems caused by an NTFS bind
mount. Immich media remains separate and visible under `storage`.

Never edit application files inside these storage folders while containers are
running. Never commit `storage`, `backups`, or `.env`.

## Requirements

- Windows 11
- Docker Desktop using the WSL2 backend and Linux containers
- At least 8 GB RAM available to Docker; more helps large video imports
- Enough free disk space for the media library, generated thumbnails, database,
  and a separate backup copy
- A Private (not Public) Windows network profile

Start Docker Desktop and wait until its engine reports that it is running.

## First installation

Run all commands from PowerShell in this repository.

### 1. Create the private environment file

```powershell
Copy-Item .env.example .env
notepad .env
```

Find the computer's LAN IPv4 address:

```powershell
Get-NetIPConfiguration |
  Where-Object IPv4DefaultGateway |
  Select-Object -ExpandProperty IPv4Address |
  Select-Object IPAddress
```

In `.env`:

1. Leave `LAN_BIND_ADDRESS=0.0.0.0`; Docker Desktop uses it to publish to the LAN.
2. Replace `CHANGE_ME_LAN_IP` with the detected address.
3. Replace `CHANGE_ME_LAN_IP_DASHED` with the same address using dashes, such
   as `192-168-1-6`.
4. Replace every password beginning with `CHANGE_ME`.
5. Use a different long password for each entry.
6. Keep `IMMICH_DB_PASSWORD` strictly alphanumeric (`A-Z`, `a-z`, `0-9`).
7. Adjust `TZ` if `Africa/Cairo` is not correct.

This PowerShell expression generates a 48-character alphanumeric secret. Run it
once for each password:

```powershell
$SecretBytes = New-Object byte[] 24
$SecretGenerator = [Security.Cryptography.RandomNumberGenerator]::Create()
$SecretGenerator.GetBytes($SecretBytes)
$SecretGenerator.Dispose()
[BitConverter]::ToString($SecretBytes).Replace('-', '')
```

`NEXTCLOUD_ADMIN_USER` and `NEXTCLOUD_ADMIN_PASSWORD` create the first
Nextcloud admin only on the first installation. Changing them later does not
change that account's password.

### 2. Prepare and validate

```powershell
New-Item -ItemType Directory -Force `
  .\storage\immich\library, `
  .\backups\nextcloud\database, `
  .\backups\immich\database

docker compose config
docker compose pull
```

`docker compose config` must finish without an error. Do not continue if it
shows a missing variable or an invalid address.

### 3. Start

```powershell
docker compose up -d
docker compose ps
```

The databases may take a minute on first start. Repeat `docker compose ps`
until `reverse-proxy`, `nextcloud`, `immich-server`, and both databases are
healthy/running.

If a container does not start:

```powershell
$ServiceName = 'nextcloud'
docker compose logs --tail 100 $ServiceName
```

Service names are listed by:

```powershell
docker compose config --services
```

### 4. Finish Nextcloud setup

Open `http://<LAN_IP>:45321` on the host computer and sign in with the
`NEXTCLOUD_ADMIN_USER` and `NEXTCLOUD_ADMIN_PASSWORD` values from `.env`.

Enable the cron background-job mode:

```powershell
docker compose exec --user www-data nextcloud php occ background:cron
docker compose exec --user www-data nextcloud php occ status
```

In Nextcloud:

1. Open **Administration settings > Users**.
2. Create a group named `family`.
3. Create a separate non-admin account for each person and add it to `family`.
4. In **Files**, create a folder named `Family`.
5. Share that folder with the `family` group and choose the needed permissions.

Each user's other files remain private unless that user shares them.

### 5. Finish Immich setup

Open `http://<LAN_IP>:12345`. The first account created is the Immich admin.
Use a strong password that is different from all database passwords.

Then:

1. Open **Administration > Users** and create one account per family member.
2. Open **Administration > Settings > Machine Learning Settings** and disable
   Smart Search and Facial Recognition for MVP1.
3. Create a test album under **Albums** and share it with another family user.
4. Set `IMMICH_ALLOW_SETUP=false` in `.env` and apply it:

```powershell
docker compose up -d
```

Use user-to-user album sharing for MVP1, not public share links.

## Home-network access

Keep the server computer awake and connected to the same home network as the
client device. From another device, use the IP placed in `.env`:

- Nextcloud: `http://<LAN_IP>:45321`
- Immich: `http://<LAN_IP>:12345`

Caddy also accepts these easier addresses on default HTTP port 80:

- Nextcloud: `http://drive.tolba`
- Immich: `http://photos.tolba`

These names are private local names, not public internet domains. They must be
created as local DNS host records on the home router or another DNS server:

```text
drive.tolba   -> <LAN_IP>
photos.tolba  -> <LAN_IP>
```

The router must distribute that DNS server to clients using DHCP. Editing the
Windows hosts file only makes the names work on that Windows computer. If the
two Wi-Fi networks are isolated from each other, the router must allow local
traffic between them; DNS and Caddy cannot bypass network isolation.

If router DNS cannot be configured, use the router-free fallback names from
`.env` instead. For the example LAN IP `192.168.1.6`, they are:

- Nextcloud: `http://drive.192-168-1-6.sslip.io`
- Immich: `http://photos.192-168-1-6.sslip.io`

The external DNS service sees the small hostname lookup, but it returns the
private LAN IP, so application traffic stays on the LAN. Some routers block
public DNS answers containing private IPs as rebinding protection; in that
case, these fallback names will not resolve. If the server LAN IP changes,
update both fallback hostname values and `NEXTCLOUD_TRUSTED_DOMAINS` in `.env`,
then run `docker compose up -d`.

The Wi-Fi network must be set to **Private**, not Public. In an Administrator
PowerShell, check and, when needed, change the profile:

```powershell
Get-NetConnectionProfile | Select-Object InterfaceAlias, NetworkCategory
Set-NetConnectionProfile -InterfaceAlias 'Wi-Fi' -NetworkCategory Private
```

Then add Private-profile-only firewall rules:

```powershell
# Replace 192.168.1.0/24 with your own home subnet if different.
Get-NetFirewallRule -DisplayName 'Family Cloud - *' | Remove-NetFirewallRule

New-NetFirewallRule `
  -DisplayName 'Family Cloud - Nextcloud' `
  -Direction Inbound -Action Allow -Protocol TCP `
  -LocalPort 45321 -RemoteAddress 192.168.1.0/24 -Profile Private

New-NetFirewallRule `
  -DisplayName 'Family Cloud - Immich' `
  -Direction Inbound -Action Allow -Protocol TCP `
  -LocalPort 12345 -RemoteAddress 192.168.1.0/24 -Profile Private

New-NetFirewallRule `
  -DisplayName 'Family Cloud - Local Proxy' `
  -Direction Inbound -Action Allow -Protocol TCP `
  -LocalPort 80 -RemoteAddress 192.168.1.0/24 -Profile Private
```

Do not create Public-profile rules. Do not forward any Family Cloud port on the
router.
If the computer's LAN IP changes, update `NEXTCLOUD_TRUSTED_DOMAINS` in `.env`,
run `docker compose up -d`, and reconnect clients with the new URL. A DHCP
reservation on the home router can keep the address stable without exposing it
to the internet.

Control the friendly local addresses without stopping either application:

```powershell
# Disable drive.tolba and photos.tolba.
docker compose stop reverse-proxy

# Enable them again.
docker compose start reverse-proxy

# Reload the proxy container.
docker compose restart reverse-proxy
```

The high-port IP addresses remain available as recovery access. Remove their
firewall rules only after the local DNS names work reliably on every intended
device.

## Acceptance tests

### Nextcloud document test

1. On another LAN device, open `http://<LAN_IP>:45321`.
2. Sign in as a non-admin family member.
3. Upload a small document to the user's private Files area.
4. Download it and confirm it opens correctly.
5. Upload a second file to the shared `Family` folder.
6. Sign in as another member and confirm that shared file is present.

### Immich phone backup test

1. Install the official Immich app on an Android or iPhone.
2. Set its server address to `http://<LAN_IP>:12345` and sign in as a non-admin.
3. Tap the cloud icon, select one small camera/test album, and enable backup.
4. Take one new photo and one short video.
5. Open/resume the Immich app and wait for both uploads to complete.
6. Open Immich in a browser and confirm both assets appear for that user.
7. Add the photo to the shared test album and confirm the invited user sees it.

For the first test, keep the phone app open. Background scheduling varies by
phone battery and data settings.

### Persistence test

```powershell
docker compose restart
docker compose ps
```

After services recover, download the same Nextcloud document and view the same
Immich photo. `docker compose restart` does not delete data.

## Daily operation

Start or apply configuration changes:

```powershell
docker compose up -d
```

Show status:

```powershell
docker compose ps
```

Stop containers without deleting data:

```powershell
docker compose stop
```

Start stopped containers:

```powershell
docker compose start
```

Stop and remove containers/networks while preserving volumes and files:

```powershell
docker compose down
```

Never add `-v` to `docker compose down`; it deletes named-volume state.

## Backup

A backup must contain all four items:

1. Nextcloud SQL dump
2. Nextcloud `data` and `html` named volumes
3. Immich SQL dump plus the entire Immich library
4. `.env` (stored securely because it contains secrets)

The following creates one consistent local backup set. It does not copy that set
to another disk; copy the completed folder to separate storage afterward.

```powershell
$BackupStamp = Get-Date -Format 'yyyy-MM-dd_HHmmss'
$BackupRoot = Join-Path (Resolve-Path .\backups) $BackupStamp
New-Item -ItemType Directory -Force `
  "$BackupRoot\nextcloud", `
  "$BackupRoot\immich"

docker compose exec --user www-data nextcloud php occ maintenance:mode --on
docker compose exec -T nextcloud-db sh -c `
  'mariadb-dump --single-transaction --default-character-set=utf8mb4 -u"$MARIADB_USER" -p"$MARIADB_PASSWORD" "$MARIADB_DATABASE" > /backup/nextcloud.sql'
docker compose exec -T immich-db sh -c `
  'pg_dump --clean --if-exists --dbname="$POSTGRES_DB" --username="$POSTGRES_USER" | gzip > /backup/immich-database.sql.gz'

docker compose stop nextcloud-cron nextcloud immich-server

tar.exe -czf "$BackupRoot\immich\immich-library.tar.gz" `
  -C .\storage\immich\library .

docker run --rm `
  --mount 'type=volume,source=family-cloud-nextcloud-html,target=/source,readonly' `
  --mount "type=bind,source=$BackupRoot\nextcloud,target=/backup" `
  --entrypoint tar nextcloud:33.0.6-apache `
  -czf /backup/nextcloud-html.tar.gz -C /source .

docker run --rm `
  --mount 'type=volume,source=family-cloud-nextcloud-data,target=/source,readonly' `
  --mount "type=bind,source=$BackupRoot\nextcloud,target=/backup" `
  --entrypoint tar nextcloud:33.0.6-apache `
  -czf /backup/nextcloud-data.tar.gz -C /source .

Copy-Item .\backups\nextcloud\database\nextcloud.sql `
  "$BackupRoot\nextcloud\nextcloud.sql"
Copy-Item .\backups\immich\database\immich-database.sql.gz `
  "$BackupRoot\immich\immich-database.sql.gz"
Copy-Item .\.env "$BackupRoot\.env"
Copy-Item .\docker-compose.yml "$BackupRoot\docker-compose.yml"

docker compose start nextcloud immich-server
docker compose exec --user www-data nextcloud php occ maintenance:mode --off
docker compose start nextcloud-cron
docker compose ps
```

Check that these files exist and are not empty:

```powershell
Get-ChildItem -Recurse $BackupRoot | Select-Object FullName, Length
```

Copy `$BackupRoot` to a different physical disk. A backup left only on the same
computer does not protect against disk failure, theft, or ransomware.

Immich also creates daily database dumps under
`storage/immich/library/backups`, but those dumps do not contain photos/videos.

## Restore

Restoring replaces current application state. Preserve the current `storage`,
`.env`, and named volumes before continuing. The commands below are intended for
a fresh replacement machine or a deliberate disaster recovery.

Use the `.env` and `docker-compose.yml` from the same backup set. Run all
commands from the restored repository directory.

### Restore Nextcloud

Set the backup folder first:

```powershell
$RestoreRoot = 'D:\path\to\backup\yyyy-MM-dd_HHmmss'
```

Stop the stack, then remove only the three Nextcloud volumes being restored:

```powershell
docker compose down
docker volume rm `
  family-cloud-nextcloud-html `
  family-cloud-nextcloud-data `
  family-cloud-nextcloud-db
docker volume create family-cloud-nextcloud-html
docker volume create family-cloud-nextcloud-data
```

Restore the file storage and application/configuration volume:

```powershell
docker run --rm `
  --mount 'type=volume,source=family-cloud-nextcloud-html,target=/target' `
  --mount "type=bind,source=$RestoreRoot\nextcloud,target=/backup,readonly" `
  --entrypoint tar nextcloud:33.0.6-apache `
  -xzf /backup/nextcloud-html.tar.gz -C /target

docker run --rm `
  --mount 'type=volume,source=family-cloud-nextcloud-data,target=/target' `
  --mount "type=bind,source=$RestoreRoot\nextcloud,target=/backup,readonly" `
  --entrypoint tar nextcloud:33.0.6-apache `
  -xzf /backup/nextcloud-data.tar.gz -C /target
```

Start a fresh database, copy in the dump, and restore it:

```powershell
Copy-Item "$RestoreRoot\nextcloud\nextcloud.sql" `
  .\backups\nextcloud\database\restore.sql
docker compose up -d nextcloud-db nextcloud-redis
docker compose exec -T nextcloud-db sh -c `
  'until mariadb-admin ping -u"$MARIADB_USER" -p"$MARIADB_PASSWORD" --silent; do sleep 2; done; mariadb -u"$MARIADB_USER" -p"$MARIADB_PASSWORD" "$MARIADB_DATABASE" < /backup/restore.sql'
docker compose up -d nextcloud nextcloud-cron
docker compose exec --user www-data nextcloud php occ maintenance:mode --off
docker compose exec --user www-data nextcloud php occ status
```

Test a known document before removing any pre-restore copies.

### Restore Immich

Immich v3 can restore a SQL dump during fresh-instance onboarding. Stop the
stack and remove only the Immich database volume:

```powershell
docker compose down
docker volume rm family-cloud-immich-db
```

Restore the media folder without immediately deleting the old copy:

```powershell
if (Test-Path .\storage\immich\library.before-restore) {
  throw '.\storage\immich\library.before-restore already exists'
}
Move-Item .\storage\immich\library .\storage\immich\library.before-restore
New-Item -ItemType Directory -Force .\storage\immich\library
tar.exe -xzf "$RestoreRoot\immich\immich-library.tar.gz" `
  -C .\storage\immich\library
```

Set `IMMICH_ALLOW_SETUP=true` in `.env`, then start Immich:

```powershell
docker compose up -d immich-db immich-valkey immich-server
```

Open `http://<LAN_IP>:12345`, choose **Restore from backup**, and upload
`$RestoreRoot\immich\immich-database.sql.gz`. After the restore succeeds, set
`IMMICH_ALLOW_SETUP=false` again and run:

```powershell
docker compose up -d
docker compose ps
```

Verify several old photos and videos before deleting `library.before-restore`.

## Troubleshooting

### Docker pipe or engine error

If a command mentions `dockerDesktopLinuxEngine` or a missing named pipe, start
Docker Desktop, confirm it is using Linux containers, and wait for the engine.

### Nextcloud says the domain is untrusted

Add the exact LAN IP or hostname to the space-separated
`NEXTCLOUD_TRUSTED_DOMAINS` value in `.env`, then run:

```powershell
docker compose up -d
$TrustedDomainIndex = @(
  docker compose exec -T --user www-data nextcloud `
    php occ config:system:get trusted_domains
).Count
docker compose exec --user www-data nextcloud `
  php occ config:system:set trusted_domains $TrustedDomainIndex `
  --value=drive.tolba
```

### Phone cannot connect

- Confirm the phone and server are on the same non-guest LAN.
- Confirm Windows calls that LAN a Private network.
- Test both URLs from the server computer first.
- Check `docker compose ps` and the Private firewall rules.
- Guest Wi-Fi/client isolation can block devices from reaching each other.

### A container is unhealthy

```powershell
docker compose ps
$ServiceName = 'nextcloud'
docker compose logs --tail 200 $ServiceName
```

Do not delete volumes as a troubleshooting step.

### Storage is filling up

```powershell
Get-PSDrive -PSProvider FileSystem
docker system df
```

Do not manually delete Immich-generated folders or Nextcloud data files. Free
space elsewhere or expand storage, then use the applications' own controls.

### Update safety

Make and verify a backup first. Read both applications' release notes. This MVP
pins Nextcloud and Immich versions; do not change image versions casually.

## Security limits

- Traffic uses plain HTTP and can be read by another device on the same LAN.
- Use unique strong passwords and trusted home devices only.
- Keep the Windows network profile Private and both firewall rules Private-only.
- Do not use router port forwarding, UPnP exposure, DMZ hosting, or public DNS.
- Keep Windows, Docker Desktop, Nextcloud, Immich, and phone apps patched through
  planned, backed-up upgrades.
- Losing both the storage and its database loses important metadata; back up both.

Official references:

- [Nextcloud Docker image documentation](https://github.com/nextcloud/docker)
- [Immich Docker Compose installation](https://docs.immich.app/install/docker-compose/)
- [Immich mobile backup](https://docs.immich.app/features/mobile-backup/)
- [Immich backup and restore](https://docs.immich.app/administration/backup-and-restore/)
