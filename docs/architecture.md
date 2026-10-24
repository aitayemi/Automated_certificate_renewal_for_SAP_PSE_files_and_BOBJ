# Architecture

## Components

| Component | Role |
|---|---|
| **Sectigo Certificate Manager** | issues the wildcard certificate; ACME endpoint `https://acme.enterprise.sectigo.com` |
| **simple-acme** (`wacs`, v2.4.0.2350, standalone/pluggable build) | ACME client, installed in `/opt/simple-acme`; stores PEM files |
| **`sapgenpse`** | SAP's PSE tool; ships with the SAP kernel; used to read and rebuild PSE files |
| **`openssl`** | validation of key/certificate, fingerprints, PKCS#8 conversion |
| **`keytool`** | BOBJ Tomcat keystore handling |
| **Ansible** | installs simple-acme, creates cron jobs, runs first-time setup |
| **Shared directory** | `/storage/tiger/acme-certs/<env>` (CIFS; training uses an NFS share). Holds the PEMs read by RHEL 7 hosts and BOBJ hosts |

### Sectigo ACME account

Created once in Certificate Manager (*Enrollment > ACME > "Universal" > Accounts*):

* Use the **Universal** ACME entry. An account created through the OV endpoint
  (`https://acme.sectigo.com/v2/OV`) fails with "domain not pre-authorized".
* Certificate profile: *Sectigo Enterprise Pro - wildcard 199 Day Only* or
  *... 99 Day Only*.
* The account provides an **EAB key identifier** and **EAB HMAC key**. They are
  needed only once, for `setup`. See [security.md](security.md).
* Domains are pre-validated at Sectigo, so simple-acme runs with
  `--validation none`. If Sectigo ever starts returning a DNS-01 challenge, set
  `ACME_VALIDATION_ARGS` (script-based validation).

## Host roles

| Role | OS | What it runs |
|---|---|---|
| Renewal host | RHEL 9 | simple-acme + `sap_pse_renew_<env>.sh` |
| Copy host | RHEL 9 (3 per landscape) | additionally the `cyclone` cron that copies PEMs to the share |
| Import host | RHEL 7 | `sap_pse_import_nfs_<env>.sh` (simple-acme does not run on RHEL 7) |
| BOBJ host | RHEL 7 | `bobj_cert_import_<env>.sh` |

The copy hosts exist only as a workaround for RHEL 7. Once all RHEL 7 application
servers are upgraded, change the default `CERTS_DIR` in the import scripts to
`/usr/sap/acme-certs`, which removes the dependency on the share, and delete the
`cyclone` copy cron entries.

Copy hosts (from the Ansible notes):

| Landscape | Copy hosts |
|---|---|
| sandbox | `cs`, `fs`, `es1pas` (`*.sandbox.tiger.edu`) |
| development | `csd`, `wd1-www`, `ed1` |
| QA | `dq1-cl1`, `eq1-d02`, `wq1-www` |
| production | `ep1-cl1`, `ep1-d02`, `wp1-www` |

## Naming conventions

* **SID** = first 3 characters of the short hostname, e.g. `es1pas` gives SID `ES1`
  and OS user `es1adm`.
* **Instance directories** searched under `/usr/sap/<SID>/`:
  `ASCS01 ERS01 C00 C01 D00 J00 SUM W00`, plus `D<NN>` derived from a hostname of
  the form `eq1-d02` (suffix after the first `-`). Only existing `sec`
  directories are used. Override with `NODES="..."`.
* **PEM file names** are produced by simple-acme's `pemfiles` store, with `*`
  replaced by `_`:

  | File | Content |
  |---|---|
  | `_.<zone>-crt.pem` | the wildcard (leaf) certificate |
  | `_.<zone>-key.pem` | its private key |
  | `_.<zone>-chain-only.pem` | the intermediates, without the leaf |
  | `_.<zone>-chain.pem` | leaf + intermediates |
  | `usertrust.pem` | the USERTrust RSA root; **not served in Sectigo's chain**, extracted manually and appended by the scripts (`EXTRA_CA_FILE`) |

  For example `*.sandbox.tiger.edu` gives `_.sandbox.tiger.edu-crt.pem`.

The shared directory also contains manually maintained files that are not produced
by simple-acme: `usertrust.pem`, `aaacscomodoca.pem`, `incommonrsaca2.pem`, the
Sectigo chain parts `chain-00.pem` / `chain-01.pem`, and a PKCS#12 file for the
portal import. Optional PK-list extras live in `pk-extra-ca.pem` and `pk-extra/`.

## Files and directories on a host

| Path | Owner | Content |
|---|---|---|
| `/opt/simple-acme/` | `<sid>adm` | simple-acme binaries, `settings.json` (`ConfigurationPath` set per user) |
| `/home/<sid>adm/.simple-acme/` | `<sid>adm`, mode 700 | simple-acme account and renewal data |
| `/usr/sap/acme-certs/` | `<sid>adm` | PEM output of simple-acme; `.last_imported.sha256` state file |
| `/usr/sap/<SID>/<NODE>/sec/*.pse` | `<sid>adm` | PSE files (SECUDIR) |
| `/usr/sap/<SID>/<NODE>/sec/backup/backup-MMDDYY-HHMMSS/` | `<sid>adm` | backups made before each update (last 5 kept per instance) |
| `/usr/sap/<SID>/<NODE>/sec/.new-<name>.pse` | `<sid>adm` | temporary new PSE while the import script runs (removed afterwards) |
| `/tmp/sap_pse_import.<user>.lock` | `<sid>adm` | `flock` lock of the import script |
| `/usr/sap/<SID>/SSL/.keystore` or `.../sap_bobj/tomcat/SSL/.keystore` | `<sid>adm` | BOBJ Tomcat keystore |

## SAP concepts used by the scripts

| Term | Meaning here |
|---|---|
| **PSE** | Personal Security Environment: a file holding a private key, the *own certificate* and a trust list |
| **Own certificate** | shown by `sapgenpse get_my_name`; this is the certificate the server presents |
| **PK list** | the PSE's list of trusted certificates, shown by `sapgenpse maintain_pk -l`. Besides CAs it can hold partner and SNC certificates |
| **`SECUDIR`** | directory holding the PSEs and `cred_v2` (SSO credentials); the scripts export it per instance |
| **`cred_v2`** | SSO credential file; lets SAP open PIN-protected PSEs without a PIN |
| **ICM soft exit** | SMICM > Administration > ICM > Exit Soft; makes the ICM reload SSL PSEs |
| **`SAPSNCS.pse`** | SNC PSE; picked up only after the affected work processes or instance restart |

## Why the PSE logic is more complicated than "import the new certificate"

* `sapgenpse import_p8` can **only create** a PSE. It refuses an existing file
  (`PSE ... already exists!`), so the private key and own certificate cannot be
  swapped in place.
* The new key and certificate therefore require a **new** PSE, and a new PSE has an
  **empty PK list**. Anything that was in the old list has to be copied over, or it
  is lost.
* `sapgenpse` has no "copy PK list" command, but `maintain_pk -l PEMlist` prints the
  list as PEM, which can be re-added one certificate at a time.

The full reasoning, with the observed behaviour of each command, is in
[import-script.md](import-script.md#what-sapgenpse-does-observed-behaviour).
