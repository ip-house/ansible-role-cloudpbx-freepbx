# FreePBX Ansible Playbook
Installs FreePBX 16 with Asterisk 20 on a fresh Ubuntu 22.04 or 24.04 minimal server, and sets up monitoring for the CloudPBX fleet.

## What gets installed
* Asterisk 20: the latest 20.x release, built from source (MP3 support, `app_macro`)
* FreePBX 16.0.40.7 with a standard set of modules
* PHP 7.4 from the `ondrej/php` PPA, Apache, MariaDB, ODBC (`odbc-mariadb`) for CDR
* phpMyAdmin 5.2.3 at `http://<server-ip>/phpmyadmin`, installed from the upstream tarball so no PHP 8 packages get pulled in
* Monitoring: prometheus-node-exporter, mysqld_exporter and promtail

Every download is checked against a pinned or published sha256.

## Requirements
* Freshly installed Ubuntu 22.04 or 24.04 minimal, reachable over SSH with sudo
* Ubuntu 20.04 is not supported (no `odbc-mariadb` package, EOL)
* Ubuntu 26.04 is not supported: FreePBX 16 needs PHP 7.4, and the `ondrej/php` PPA does not build for 26.04
* Ansible with the required collections:
  ```sh
  ansible-galaxy collection install -r requirements.yml
  ```

## Usage
```sh
ansible-playbook -i '<server-ip>,' -u ubuntu cloudpbx-freepbx.yaml
```
The playbook asks for:
* **Customer name**: lowercase letters, digits and dashes. The hostname becomes `<name>-pbx.cloudpbx.rs`
* **DB root password**: press Enter to use `DB_ROOT_PASS` from the environment
* **Provider label** for Loki logs (default `nc`)

To run without prompts, pass the answers with `-e`:
```sh
export DB_ROOT_PASS='strong-password'
ansible-playbook -i '<server-ip>,' -u ubuntu \
  -e customer_name=acme -e monitoring_provider=nc -e db_root_pass="$DB_ROOT_PASS" \
  cloudpbx-freepbx.yaml
```
A full run takes about 30-40 minutes, most of it compiling Asterisk. The server reboots after the system upgrade (only if Ubuntu asks for it) and once more after the first FreePBX install. When the playbook finishes, the FreePBX web UI is at `http://<server-ip>/admin`.

The playbook can be re-run safely: a second run changes nothing.

## Settings
Change these in `cloudpbx-freepbx.yaml`:

| Variable | Default | Meaning |
|---|---|---|
| `pbx_domain` | `cloudpbx.rs` | Domain part of the hostname |
| `ast_country_code` | `381` | Country code for the libvpb1 package |
| `fpbx_auto_module_updates` | `disabled` | FreePBX module auto-upgrades (`enabled`, `emailonly`, `disabled`) |
| `fpbx_auto_security_updates` | `emailonly` | Automatic FreePBX security updates (`enabled`, `emailonly`) |
| `fpbx_notification_email` | empty | Recipient for FreePBX update mails; empty means no mails are sent |

Loki servers, exporter versions and checksums live in `roles/monitoring/vars/main.yml`.

## What else gets configured
* **FreePBX updates**: no automatic module upgrades and no update mails. FreePBX rebuilds the `asterisk` crontab from its update settings on every `fwconsole reload` and every *Apply Config*. Editing those cron lines by hand does not last, so the playbook sets the FreePBX settings instead. One `ma listonline --sendemail` check line always remains; with no recipient it sends nothing.
* **Hostname**: set to `<customer>-pbx.cloudpbx.rs` and added to `/etc/hosts`. The same name is used as the FreePBX system identity.
* **Monitoring**:
  * node_exporter on port 9100 and mysqld_exporter on port 9104 listen only on the server IP of the default route (`monitoring_host_ip`), which is the IP Prometheus scrapes
  * mysqld_exporter logs in to MariaDB through `unix_socket`, so it needs no password
  * promtail ships `/var/log/asterisk/full` to Loki (`loki.cloudoffice.local`, added to `/etc/hosts`). Its own ports listen only on 127.0.0.1, and it reads the logs through a read-only ACL
* **Backups** are not configured; set them up in the FreePBX Backup module after the install.

## Running on servers installed with an older version
* The marker for the FreePBX module install moved to `/var/lib/cloudpbx/`. On an existing server the module loop would run once more, including `fwconsole restart`, which drops calls. To skip it, run this on the server first:
  ```sh
  sudo mkdir -p /var/lib/cloudpbx && sudo touch /var/lib/cloudpbx/fpbx-modules.ok
  ```
  Then run `sudo fwconsole chown` once, so `fwconsole` is root-owned again.
* Remove a phpMyAdmin installed from the Ubuntu package before the run, otherwise linking `/usr/share/phpmyadmin` fails.
* Webmin is no longer installed, but an existing installation is not removed.

## Known issues
* MariaDB listens on `0.0.0.0`, although FreePBX only uses the database locally.
* The task *Disable linux socket root login* fails on MariaDB 10.4+ and is ignored.

## Validation
```sh
ansible-playbook --syntax-check -i '127.0.0.1,' -e customer_name=x -e db_root_pass=x -e monitoring_provider=nc cloudpbx-freepbx.yaml
ansible-lint cloudpbx-freepbx.yaml
pre-commit run --all-files
```
Test full runs on a disposable Ubuntu host.
