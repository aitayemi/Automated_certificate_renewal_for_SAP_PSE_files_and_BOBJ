# `bobj_cert_import_<env>.sh` - BOBJ Tomcat keystore update

Updates the `tomcat` entry of the BusinessObjects Tomcat keystore from the PEM files
in the shared directory. It is written for BOBJ but works for any Tomcat that keeps a
private key under an alias in a Java keystore.

> The script **does not check the certificate's CN**: it imports whatever is in
> `CERT_DIR`. Use the script of the correct environment on the correct hosts only.

## Hosts

| Env | Hosts | OS user | Keystore |
|---|---|---|---|
| sbx | `bsdpas` | `bsdadm` | `/usr/sap/BSD/sap_bobj/tomcat/SSL/.keystore` |
| dev | `bodpas` | `bodadm` | `/usr/sap/BOD/sap_bobj/tomcat/SSL/.keystore` |
| qa | `jq1bobjcl1`, `jq1bobjcl2` (2-node cluster) | `jq1adm`, `jq2adm` | `/usr/sap/<SID>/SSL/.keystore` |
| prod | `jp1bobjcl1`, `jp1bobjcl2` (2-node cluster) | `jp1adm`, `jp2adm` | `/usr/sap/<SID>/SSL/.keystore` |

Sandbox and development keep the keystore under `sap_bobj/tomcat/SSL`; QA and
production under `<SID>/SSL`. That path is the only difference besides the
environment's certificate names and keystore password variable.

## Scheduling

The job is in the **`cyclone`** user's crontab and runs as `<sid>adm` through sudo:

```
0 2 * * * /bin/sudo -u <sid>adm JAVA_HOME=<path to JRE> /storage/tiger/acme-certs/certscripts/bobj_cert_import_<env>.sh >> /tmp/update_tomcat_cert.log 2>&1
```

Daily at 02:00. It is harmless when nothing has changed (it exits `0` after the
"already matches" check). `JAVA_HOME` contains the exact OpenJDK build, so it changes
with Java updates. In practice the script calls `/usr/bin/keytool` (see below), so
check the cron lines after a Java update, but a stale path is unlikely to break the job.

## Configuration (top of the script)

| Variable | Meaning |
|---|---|
| `CERT_DIR`, `CERT_FILE`, `KEY_FILE`, `CHAIN_FILE` | share directory and the three PEM files (`CHAIN_FILE` is the intermediates only) |
| `ALIAS` | keystore alias, `tomcat` |
| `KS_PASS` | keystore password (also used for the temporary PKCS#12). **Do not publish it**, see [security.md](security.md) |
| `KEYTOOL` | defaults to `/usr/bin/keytool`; can be overridden in the environment |
| `SSL_DIR`, `KEYSTORE` | derived from the SID (see the table above) |

The SID is the first three characters of the hostname (lower case for the user,
upper case for the `/usr/sap` directory). The script must run as `<sid>adm`.

## Flow

1. **Checks:** run as `<sid>adm`; `keytool` and `openssl` exist; keystore, certificate,
   key and chain readable (an unreadable file usually means the share is not mounted);
   certificate not expired; private key matches the certificate.
2. **Skip if current:** the SHA256 fingerprint of the certificate on the share is
   compared with the keystore entry. Equal means exit `0`, nothing to do.
3. **Backup:** `cp -p .keystore .keystore.<MMDDYYYYHHMMSS>`. No pruning is done;
   backups accumulate (one per renewal).
4. **Build a temporary PKCS#12** from key, certificate and chain, in a private
   temporary directory (removed on exit). The password is passed through the
   environment (`KS_PASS_ENV`), not on the command line.
5. **Replace the entry:** `keytool -delete -alias tomcat`, then
   `keytool -importkeystore ... -destalias tomcat`.
6. **Verify:** the keystore entry's SHA256 must equal the share's. Any failure
   restores the backup.
7. The message `restart Tomcat for the new certificate to take effect` is logged.
   **Nothing restarts Tomcat.**

Exit codes: `0` success or already current, `1` any error (the keystore is restored
where that applies).

Log: `/tmp/update_tomcat_cert.log` on each host.

## Restarting Tomcat

On a clustered BOBJ do **one node at a time** so the other keeps serving. Per node:

1. stop Tomcat (`tomcatshutdown.sh`) and the servers (`stopservers`) in
   `/usr/sap/<SID>/sap_bobj`;
2. start the servers (`startservers`) and Tomcat (`tomcatstartup.sh`);
3. check the CMC URL (`https://<host>:8443/BOE/CMC`) and the served certificate with
   `openssl s_client -connect <host>:8443 -showcerts`;
4. repeat on the second node.

Sandbox and development are single-node; QA and production have two nodes.

## Maintenance notes

* The backup file name comment in the script says `MMDDYYYYSS`; the real format is
  `%m%d%Y%H%M%S`.
* The block that searches for `keytool` in `JAVA_HOME`, `PATH` and the BOBJ JVM only
  runs when `KEYTOOL` is empty, which cannot happen because it defaults to
  `/usr/bin/keytool`. To use the bundled JVM, set `KEYTOOL=...` in the cron line.
* `CHAIN_FILE` holds the intermediates; the keystore entry therefore has leaf plus
  intermediates but not the USERTrust root, which is normal for a server entry.

## Second cluster nodes (verify)

The script derives the user and the SSL path from the **hostname prefix**. On the
second nodes (`jq1bobjcl2`, `jp1bobjcl2`) the cron job runs as `jq2adm` / `jp2adm`
and the instance directory is `JQ2` / `JP2`, but the hostname still starts with
`jq1` / `jp1`. The script would then compute `SIDADM=jq1adm` and the
`/usr/sap/JQ1/SSL` keystore path, fail the "must be run as" check and exit `1`.

This was derived from reading the script and the cron lines, **not from a log**.
Check `/tmp/update_tomcat_cert.log` on the second nodes. If it shows
`must be run as jq1adm`, set the SID explicitly for those hosts (a small change to
take `SID_LOWER` from an environment variable), or run the job through a
per-host wrapper.

## Other system types

| System | Procedure |
|---|---|
| **Portal** | NWA > *SSL Configuration*: import the certificate (PKCS#12), choose to restart the ICM when asked. There can be only one `PRIVATE KEY` entry, so delete the old wildcard first. Check with `openssl s_client -connect localhost:<HTTPS port> -showcerts` |
| **Content Server** | restart the service to load the renewed certificate (`tiger service restart`) |
