# Runbook

Placeholders: `<env>` = `sbx|dev|qa|prod|train`, `<sid>` = first three characters of
the hostname in lower case, `<SID>` = the same in upper case, `<NODE>` = instance
directory such as `D00`.

## Contents

1. [Prerequisites](#prerequisites)
2. [One-time Sectigo account](#one-time-sectigo-account)
3. [Install simple-acme](#install-simple-acme)
4. [One-time ACME setup](#one-time-acme-setup)
5. [Cron jobs](#cron-jobs)
6. [First run](#first-run)
7. [Verifying a PSE](#verifying-a-pse)
8. [Activating the new certificate](#activating-the-new-certificate)
9. [Rolling back](#rolling-back)
10. [Routine operations](#routine-operations)
11. [Retiring the RHEL 7 workaround](#retiring-the-rhel-7-workaround)

## Prerequisites

* Ansible access with `-b` (sudo) to the hosts; `cyclone` for the copy and BOBJ jobs.
* The simple-acme standalone zip (`simple-acme.v2.4.0.2350.linux-x64.pluggable.zip`).
* `<sid>adm` is allowed to use cron: the line `+ : ALL : CRON` must be in
  `/etc/security/access.conf` before `- : ALL : ALL`.
* The shared directory is mounted on every host that reads from it.
* A Sectigo ACME account (next section).

Allow cron for all users (idempotent):

```
ansible all -b -m lineinfile -a 'path=/etc/security/access.conf line="+ : ALL : CRON" insertbefore="^- : ALL : ALL$" state=present backup=yes'
```

## One-time Sectigo account

1. Sign in to Certificate Manager with SSO (administrator rights needed).
2. *Enrollment > ACME > "Universal" > Accounts*, click **+**, give the account a name
   (for example `sap-tiger-universal`).
3. Certificate profile: *Sectigo Enterprise Pro - wildcard 199 Day Only* or *... 99 Day
   Only*. Save.
4. Record the account's **EAB key identifier** and **EAB HMAC key** in your password
   manager. Do not write them into scripts or notes (see [security.md](security.md)).

Do not use the OV endpoint (`https://acme.sectigo.com/v2/OV`); it produces
"domain not pre-authorized" errors.

## Install simple-acme

`install_simple_acme.sh` is run as root through Ansible. It:

* verifies that `<sid>adm` exists (so the hostname must map to an SAP SID user);
* unpacks the zip into `/opt/simple-acme`, makes `wacs` executable and links
  `simple-acme` to it;
* creates `settings.json` from `settings_default.json` and sets `ConfigurationPath` to
  `/home/<sid>adm/.simple-acme`;
* creates `/usr/sap/acme-certs` and `~/.simple-acme` (mode 700) owned by `<sid>adm`;
* checks that `<sid>adm` can execute, and write to, everything it needs.

```
ansible HOST -m copy  -a 'src=simple-acme.v2.4.0.2350.linux-x64.pluggable.zip dest=/tmp/simple-acme.2402350.linux.zip mode=0644'
ansible HOST -m copy  -a 'src=install_simple_acme.sh dest=/tmp/install_simple_acme.sh mode=0700' -b
ansible HOST -m shell -a '/tmp/install_simple_acme.sh' -b
```

Test on one host first (`-l`), and run without `-o` so error output is visible.
Success prints `OK: <host>: simple-acme installed. SID=... user=... ConfigurationPath=...`.
No ACME credentials belong in this script.

## One-time ACME setup

Registers the ACME account and creates the renewal for the environment's wildcard.
Run once per host that renews (RHEL 9), as `<sid>adm`, passing the EAB credentials in
the environment:

```
ansible <group> -m shell -a '
ADM=$(hostname -s | cut -c1-3 | tr "[:upper:]" "[:lower:]")
sudo -u "${ADM}adm" \
EAB_KID="<EAB_KID>" \
EAB_HMAC="<EAB_HMAC>" \
ACME_BASEURI="https://acme.enterprise.sectigo.com" \
/storage/tiger/acme-certs/certscripts/sap_pse_renew_<env>.sh setup
' -o
```

Ansible group names used so far: `sandbox`, `development`, `qa`, `production`,
`training`. Success ends with `Setup complete.` The PEM files appear in
`/usr/sap/acme-certs`.

## Cron jobs

All jobs use the idempotent pattern "add if the script is not already in the
crontab". Run through Ansible with `-b`; the user is `<sid>adm` unless stated.

| Job | Entry | User | Hosts |
|---|---|---|---|
| Renew | `0 1 * * * /storage/tiger/acme-certs/certscripts/sap_pse_renew_<env>.sh` | `<sid>adm` | RHEL 9 |
| Copy to share | `cp -u --preserve=timestamps /usr/sap/acme-certs/*.pem /storage/tiger/acme-certs/<dir>/ >> /home/cyclone/acme_copy.log 2>&1` at minute 15, hour 1 | `cyclone` | 3 RHEL 9 hosts per landscape |
| Import | `30 1 * * * /storage/tiger/acme-certs/certscripts/sap_pse_import_nfs_<env>.sh` | `<sid>adm` | RHEL 7 |
| BOBJ Tomcat | see [bobj-tomcat.md](bobj-tomcat.md#scheduling) | `cyclone` (sudo to `<sid>adm`) | BOBJ hosts |

Example (renew job, sandbox):

```
ansible sandbox -b -m shell -a '
SIDADM=$(hostname -s | cut -c1-3 | tr "[:upper:]" "[:lower:]")adm
sudo -u "$SIDADM" bash -c "
crontab -l 2>/dev/null | grep -Fq /storage/tiger/acme-certs/certscripts/sap_pse_renew_sbx.sh ||
(
  crontab -l 2>/dev/null
  echo '\''0 1 * * * /storage/tiger/acme-certs/certscripts/sap_pse_renew_sbx.sh'\''
) | crontab -
"
'
```

The same pattern, with the script name and time changed, installs the import job.
The Ansible notes apply these entries to whole inventory groups; **restrict each job to
the hosts that need it** (renewal on RHEL 9, import on RHEL 7) with `-l`.

Copy job through Ansible's cron module (per host, directory `sbx|dev|qa|prod`):

```
ansible <host> -b -m ansible.builtin.cron -a 'name="copy acme certs to share" user=cyclone minute=15 hour=1 job="cp -u --preserve=timestamps /usr/sap/acme-certs/*.pem /storage/tiger/acme-certs/<dir>/ >> /home/cyclone/acme_copy.log 2>&1"'
```

The copy job must run after the renewal (01:00) and before the import (01:30). The
import script waits two minutes for freshly written files.

**Tip:** the cron lines above send output to cron mail. Append
`>> /tmp/sap_pse_import.log 2>&1` (or similar) to the import and renew entries so
there is a log to read.

## First run

```
# as <sid>adm
/storage/tiger/acme-certs/certscripts/sap_pse_renew_<env>.sh --dry-run     # renew hosts: lists PSEs that would change
/storage/tiger/acme-certs/certscripts/sap_pse_import_nfs_<env>.sh --dry-run # import hosts
```

A dry run prints `DRYRUN <node>/<pse> would be updated` for every PSE that holds the
wildcard and `SKIP ... (own cert is not CN=...)` for the rest. PSEs that need a PIN
(for example `sap_system_pki_instance.pse`) show up as skipped with
`Missing PIN/Passphrase`, which is expected.

Then run for real, on one instance first:

```
NODES=D00 /storage/tiger/acme-certs/certscripts/sap_pse_import_nfs_<env>.sh
```

Take your own copy of the `sec` directory beforehand the first time.

## Verifying a PSE

```
export SECUDIR=/usr/sap/<SID>/<NODE>/sec

# 1. own certificate: expect the new serial and the Sectigo issuer
sapgenpse get_my_name -p $SECUDIR/SAPSSLS.pse

# 2. PK list: one block per entry
sapgenpse maintain_pk -l -p $SECUDIR/SAPSSLS.pse | grep -E '^Element|^  Subject:|SHA1'

# 3. what the share offers
openssl x509 -in /storage/tiger/acme-certs/<dir>/_.<zone>-crt.pem -noout -subject -issuer -serial -dates
```

What to expect in the PK list after an update: the new wildcard, the Sectigo
intermediate and root, the USERTrust root, and **every unexpired entry that was there
before** (a wildcard from the previous issuer, DigiCert/VeriSign roots, partner
certificates, ...). The expired ones are in `<backup-dir>/<name>.pklist.pem`.

Check the **real** PSE under `/usr/sap/<SID>/<NODE>/sec`, not an old copy in `/tmp`.

## Activating the new certificate

| What | How |
|---|---|
| SSL PSEs (`SAPSSLS`, `SAPSSLC`, ...) | SMICM > Administration > ICM > **Exit Soft**, or run the script with `RESTART_ICM=1` (sends SIGINT to the ICM; connections of that instance are briefly interrupted) |
| `SAPSNCS.pse` | work processes or instance restart |
| BOBJ Tomcat | manual restart, see [bobj-tomcat.md](bobj-tomcat.md#restarting-tomcat) |
| Portal | import via NWA, restart ICM when prompted |
| Content Server | restart the service |

Check what a server presents: `openssl s_client -connect <host>:<port> -showcerts`.

## Rolling back

**PSE.** The newest backup is under `/usr/sap/<SID>/<NODE>/sec/backup/`:

```
cd /usr/sap/<SID>/<NODE>/sec
ls -ldt backup/backup-*                              # newest first
cp -p backup/backup-<timestamp>/SAPSSLS.pse .        # restore one PSE
cp -p backup/backup-<timestamp>/cred_v2 .            # only if the credential file changed
```

then soft-exit the ICM. Nothing else needs to be undone. Only five backups per
instance are kept; take your own copy of anything you need to keep longer.

**BOBJ keystore.**

```
cp -p /usr/sap/<SID>/.../.keystore.<MMDDYYYYHHMMSS> /usr/sap/<SID>/.../.keystore
```

then restart Tomcat. The scripts restore automatically on their own failures.

## Routine operations

| Task | Command |
|---|---|
| Force a new certificate at the CA and re-import (renew host) | `sap_pse_renew_<env>.sh --force` |
| Re-import the current certificate without a CA request (renew host) | `sap_pse_renew_<env>.sh --reimport` |
| Rebuild PSEs even if current (import host) | `sap_pse_import_nfs_<env>.sh --force` (use for a PSE whose PK list is incomplete) |
| Restrict a run to one instance | `NODES=D00 ...` |
| Restrict the renew script to one PSE | `PSE_ONLY="SAPSSLS.pse" ...` |
| Let the script restart the ICM | `RESTART_ICM=1 ...` |
| Check certificate expiry on the share | `openssl x509 -in <crt> -noout -enddate` |
| Add a certificate that every PK list must contain | put the PEM into `<share>/pk-extra/` and run with `--force` |
| Add a new host | install simple-acme (RHEL 9) or install the import script and cron (RHEL 7); first `--dry-run` |
| Add a new landscape | copy the scripts, change `CERT_CN`, `CERTS_DIR`, `ACME_FRIENDLY_NAME`; create the share directory; run `setup` |
| Upgrade simple-acme | repeat *Install simple-acme* with the new zip; check `settings.json` kept its `ConfigurationPath` |

## Retiring the RHEL 7 workaround

When no RHEL 7 application server is left:

1. in the import scripts, change the default
   `CERTS_DIR="${CERTS_DIR:-/storage/tiger/acme-certs/<dir>}"` to
   `CERTS_DIR="${CERTS_DIR:-/usr/sap/acme-certs}"`;
2. delete the `cyclone` copy cron entries on the copy hosts (see [architecture.md](architecture.md#host-roles));
3. keep the share only for BOBJ if those hosts still read from it.

This is also the right moment to apply the recommendation in
[renew-script.md](renew-script.md#recommended-fix): let the import script perform all PSE updates.
