# Troubleshooting

First step for any problem with the PSE scripts: **read the log of the failing run
from the top** and find the first `FAILED`, `WARN` or `ERROR`. The lines just above
it name the check that failed. Log format and meaning:
[import-script.md](import-script.md#reading-the-log).

## Quick diagnostics

```bash
export SECUDIR=/usr/sap/<SID>/<NODE>/sec                       # silences the "SECUDIR is not set" warning
sapgenpse get_my_name -p $SECUDIR/<pse>                        # own certificate (subject, issuer, serial, expiry)
sapgenpse maintain_pk -l -p $SECUDIR/<pse>                     # PK list (one block per entry, with SHA1)
sapgenpse maintain_pk -p $SECUDIR/<pse> -l PEMlist 2>&1 \
  | grep -c 'BEGIN CERTIFICATE'                                # how many certificates the export yields
grep -c add_pem_to_pklist <path-to-script>                     # >0 means the current import script version
ls -l /usr/sap/<SID>/<NODE>/sec /usr/sap/<SID>/<NODE>/sec/backup
```

## Is this the right script version?

Output that does not start with `YYYY-MM-DD HH:MM:SS [<host>]` timestamps (for
example "Validating certificate and key...", "Creating temporary PSE") comes from
a different script than the one documented here. Check the file that cron or you
actually run: `grep -c add_pem_to_pklist <script>` and compare with the repository.
A stale or hand-edited copy on the share has already caused one round of confusion.

## PSE import and renew scripts

| Message or symptom | Cause | What to do |
|---|---|---|
| `import_p8: ... you didn't supply any certificates!` | `import_p8` called without `-c` | old or edited script; `import_p8` always needs `-c <leaf + chain>` |
| `import_p8: ... certificate chain is incomplete, need certificate of 'CN=...'` | `-c` file has no intermediates (leaf only) | `-c` must be leaf followed by `-chain-only.pem`; check that file contains the intermediate |
| `import_p8: You must supply the name of a p8-file!` | the script passes no key file to `import_p8` | wrong or modified script on the share |
| `import_p8: ERROR -- PSE ... already exists!` | `import_p8` never overwrites. In the current script this cannot happen because it builds `.new-<name>` | if you ran `import_p8` by hand, delete the target first. If the script shows it, a leftover `.new-<name>` may exist: remove it |
| `SKIP ... Couldn't open PSE ... (Missing PIN/Passphrase, no credentials found)` | PIN-protected PSE, no SSO credential for this user (for example `sap_system_pki_instance.pse`) | expected; such PSEs are not touched. Set `PSE_PIN` only if you really want them handled |
| `SKIP ... (own cert is not CN=...)` | the PSE holds another certificate | expected for PSEs of other products or environments |
| `OK ... already holds the share's certificate` | the own certificate has the share's serial | nothing to do. If the PK list is incomplete, run `--force` |
| `WARN ... cannot read expiry from sapgenpse output` | `get_my_name` text not as expected | check the output by hand; format change needs a parser fix; `--force` updates anyway |
| `file changed Ns ago - may still be copied` | share files younger than `MIN_AGE_SECONDS` (120) | wait and rerun; the run exits `0` |
| `another instance is already running (lock ...)` | `flock` held by a running or hung run | check `ps`; remove nothing: the lock is released when the process ends |
| `Do not run as root` | | run as `<sid>adm` |
| `Certificate and private key on the share do not match` | the three files are from different renewals, or a copy is in progress | wait for the copy job; compare modification times; rerun the renewal |
| `Certificate on the share is expired or expires within a day` | | check the renewal (`simple-acme` log, Sectigo) |
| `No SEC directory found for any of: ...` | hostname does not map to the SID, or no instance on this host | check `hostname -s`; set `NODES` or `HOST_SHORT` |
| `sapgenpse not found for instance` | not in the instance `exe`, `SYS/exe/run` or `PATH` | set `SAPGENPSE=/path/to/sapgenpse` |
| `WARNING: <key> is accessible to 'other' users` | file mode on the share | `chgrp <group> <key>; chmod 640 <key>` ([security.md](security.md)) |
| `PK list entry of X missing in the PEMlist export` | `maintain_pk -l PEMlist` did not return every entry | run the command by hand (see Quick diagnostics); possible causes: PIN, changed output format, truncated output. The PSE is left untouched |
| `sapgenpse accepted but did not keep (EXPIRED, ...)` | expired certificate silently refused | expected for expired old entries; they are archived |
| `expired entry not carried over` | old expired PK entry | expected; informational |
| `lost a pre-existing entry: SHA1 ..., expires ..., <subject>` | an unexpired old entry is missing from the new PSE | the line gives SHA1, expiry and subject. Look above for `not added, continuing` for the same subject (the `sapgenpse` message is there). If the certificate is a duplicate of one that was added, check the fingerprints. As a last resort `PK_PRESERVE_STRICT=0`, then re-add the entry from `<backup>/<name>.pklist.pem` |
| `PK list of X is missing: <subject> (SHA1 ...)` | wildcard or a chain certificate did not stay in the list | look above for `not added` or `did not keep`; `PKLIST_STRICT=0` only to get unblocked |
| `own certificate serial is 'A', expected 'B'` | the own certificate is not the new one | the script then tries `import_own_cert`; if that fails too, check chain completeness and the key/certificate pair |
| `FAILED ... the new PSE was discarded, the original PSE is unchanged` | a check failed before the swap | the original PSE is intact; fix the cause named above it and rerun |
| `FAILED ... after the new PSE was installed - restoring` | `seclogin` or another step after the swap failed | the backup was restored; check `PSE_PIN` and the `sec` directory permissions |
| `get_my_name` still shows the old certificate | (1) the run failed, see the log; (2) you looked at a copy, for example in `/tmp`; (3) the PSE was skipped as current | check the log for `FAILED`; query the real path under `sec`; `--force` |
| PSE has the new certificate but the application serves the old one | the process did not reload | ICM soft exit; instance restart for `SAPSNCS.pse` |
| PK list has the new certificate but lost old partner certificates (renew host) | the renew script recreates PSEs without keeping the list | see [renew-script.md](renew-script.md#known-gap-pk-list-handling); restore from `<backup>/<name>.pklist.pem` |
| `Warning: Environment variable SECUDIR is not set` (by hand) | harmless when `-p` has a full path | `export SECUDIR=...` |

### Re-adding a lost PK list entry by hand

```
cd /usr/sap/<SID>/<NODE>/sec/backup/backup-<timestamp>
awk '/BEGIN CERTIFICATE/{n++} n{print > ("c" n ".pem")}' <name>.pklist.pem   # split the archive
for f in c*.pem; do echo "== $f"; openssl x509 -in $f -noout -subject -enddate -fingerprint -sha1; done
sapgenpse maintain_pk -a c<N>.pem -p /usr/sap/<SID>/<NODE>/sec/<pse>
```

Run the `sapgenpse` command as `<sid>adm`. Remember that expired certificates are
refused silently.

## renew script and simple-acme

| Message or symptom | Cause | What to do |
|---|---|---|
| `simple-acme has no renewal configured ... No scheduled renewals found` | `setup` was never run for this user, or simple-acme cannot write its configuration | run `sap_pse_renew_<env>.sh setup` as the cron user; check `ConfigurationPath` in `/opt/simple-acme/settings.json` and the owner of `~/.simple-acme` |
| `Set EAB_KID and EAB_HMAC for setup` | credentials not in the environment | pass them for the `setup` run only |
| `Expected file missing: .../_.<zone>-crt.pem` | PEM names differ from the CN | check `ls /usr/sap/acme-certs`; set `PEM_PREFIX` |
| `Certificate unchanged since last import - nothing to do.` | the certificate hash equals `.last_imported.sha256` | normal; `--reimport` to push the current one again |
| "domain not pre-authorized" from Sectigo | account created through the OV endpoint | create a **Universal** ACME account |
| simple-acme: `Running as limited user, some options including autorenewal disabled` / `Unable to configure cronjob` | simple-acme is not root | harmless; scheduling is done by the `<sid>adm` crontab |
| simple-acme does not start on an old host | glibc too old | `ACME_LD_LIBRARY_PATH` to a newer runtime, or run on RHEL 9 |
| Chain misses the USERTrust root | Sectigo does not serve it | keep `usertrust.pem` and `EXTRA_CA_FILE` |
| Cron job never runs | `<sid>adm` not allowed to use cron | `/etc/security/access.conf` needs `+ : ALL : CRON` |

## BOBJ Tomcat script

| Message or symptom | Cause | What to do |
|---|---|---|
| `must be run as <sid>adm (current user: ...)` | SID from the hostname does not match the user | on second cluster nodes see [bobj-tomcat.md](bobj-tomcat.md#second-cluster-nodes-verify) |
| `cannot read <file> (is the CIFS share mounted?)` | share unmounted or file missing | mount the share; check names |
| `certificate in <file> is expired` | renewal failed | fix the renewal |
| `private key does not match certificate` | files from different renewals | wait or rerun the renewal |
| `Keystore 'tomcat' entry already matches ...` | nothing to do | normal |
| `ERROR: import failed` / `verification failed` | keytool error | the keystore was restored; read the keytool message above |
| Tomcat still presents the old certificate | no restart | restart Tomcat |
| `keytool not found` | `/usr/bin/keytool` missing | set `KEYTOOL=...` |

## Openssl and the PKCS#12 on the share

`openssl pkcs12 -in <file>.p12 ...` fails on OpenSSL 3 with
`unsupported ... Algorithm (RC2-40-CBC)` for older PKCS#12 files. Add `-legacy`
(`openssl pkcs12 -legacy -in <file>.p12 -clcerts -nokeys | openssl x509 -noout -subject -issuer -serial -dates`),
or read the PEM files instead. The PSE scripts do not use that file.
