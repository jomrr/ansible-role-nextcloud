# Ansible Role: nextcloud

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-nextcloud)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-nextcloud)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-nextcloud)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-nextcloud/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-nextcloud/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-nextcloud/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-nextcloud/actions/workflows/main.yml?query=branch%3Amain)

Install Nextcloud from its verified release archive and configure an application
on a prepared PHP and database stack.

## Purpose

Install the current Nextcloud server archive on a prepared PHP, webserver and
database stack. Initialize the application, manage site settings and run
background jobs every five minutes. Unchanged subsequent runs preserve the
installed release and are idempotent.

## Scope

### Managed

- Download latest.tar.bz2 over HTTPS and verify it against the publisher's
  latest.tar.bz2.sha256 before extraction.
- Application ownership, a private data directory and initial database schema
  and administrator creation through occ.
- Trusted domains, the public CLI URL, additional system settings and cron
  scheduling.
- Installation and activation of explicitly requested Nextcloud apps, preserving
  other apps and installed versions.
- Persistent SELinux contexts for application and writable paths, plus
  explicitly configured SELinux booleans.
- Archive extraction tools, ACL support for privilege changes and the cron
  service.

### Not Managed

- PHP, PHP extensions, runtime accounts, webserver virtual hosts, TLS
  certificates and database provisioning.
- Redis, mail services, firewall rules, application upgrades, backups and data
  migrations.

## Requirements

- PHP CLI and a web runtime meeting the downloaded release's requirements,
  including the database driver and required extensions. Both runtimes must use
  compatible PHP versions and settings.
- Enable ctype, curl, DOM, fileinfo, GD, mbstring, OpenSSL, POSIX, session,
  SimpleXML, XMLReader, XMLWriter, zip and zlib, plus pdo_mysql or pdo_pgsql.
  Check actual loaded modules in both CLI and FPM; distribution package names
  differ. Set memory_limit to at least 512M for both runtimes. PHP-FPM pool
  overrides can supersede php.ini. Install intl, sodium, sysvsem and CLI pcntl
  where appropriate; selected apps may need further extensions.
- An empty MySQL/MariaDB or PostgreSQL database and an account with privileges
  on it. The database must meet Nextcloud's version and configuration
  requirements. Remote database transport must be secured separately.
- A configured HTTPS virtual host serving nextcloud_install_dir, with
  Nextcloud's access restrictions and routing rules. The webserver and PHP
  account must be able to read the application through nextcloud_group. Keep the
  virtual host inaccessible to clients until initial installation completes.
- With Apache proxy_fcgi and cgi.fix_pathinfo=0, prevent PATH_TRANSLATED from
  overriding SCRIPT_FILENAME for WebDAV paths. The tested PHP handler uses
  ProxyFCGISetEnvIf "true" !PATH_TRANSLATED while retaining the PHP role's
  path-handling restrictions.
- The data directory must be outside all web-accessible document roots; the role
  checks separation from its application directory.
- Set NEXTCLOUD_CONFIG_DIR=/etc/nextcloud in the PHP web runtime, or use the
  value of nextcloud_config_dir. With PHP-FPM, configure
  env[NEXTCLOUD_CONFIG_DIR] in the application pool. The role supplies this
  environment variable to occ and cron.
- PHP service confinement must permit access to the configured paths. For
  example, openSUSE's PHP-FPM service uses ProtectSystem=full, which makes /etc
  read-only. Create the configuration directory and permit
  ReadWritePaths=/etc/nextcloud in a service override before restarting that
  runtime. Retain the remaining confinement; adapt allowances for custom paths
  and stricter service sandboxes.
- The target must reach download.nextcloud.com and, when installing apps, the
  Nextcloud App Store and app download hosts over HTTPS.
- The PHP, HTTP server and database roles listed as dependencies prepare
  Molecule fixtures only; they are not automatically applied by this role.
  Configure the production stack separately and adapt nextcloud_sebooleans to
  its services and storage.

## Dependencies

```yaml
collections:
  - name: ansible.posix
    version: '>=2.0.0'
  - name: community.general
    version: '>=12.0.0'
  - name: community.crypto
    version: '>=3.0.0'
  - name: ansible.mariadb
    version: '>=6.0.0,<7.0.0'
  - name: community.postgresql
    version: '>=4.0.0,<5.0.0'
roles:
  - name: jomrr.php
    src: https://github.com/jomrr/ansible-role-php.git
    scm: git
    version: main
  - name: jomrr.httpd
    src: https://github.com/jomrr/ansible-role-httpd.git
    scm: git
    version: main
  - name: jomrr.nginx
    src: https://github.com/jomrr/ansible-role-nginx.git
    scm: git
    version: main
  - name: jomrr.mariadb
    src: https://github.com/jomrr/ansible-role-mariadb.git
    scm: git
    version: main
  - name: jomrr.postgresql
    src: https://github.com/jomrr/ansible-role-postgresql.git
    scm: git
    version: main
```

## Role Variables

### `nextcloud_install_dir`

Type: `path`. Required: `false`.

Absolute application directory; configure the webserver document root
separately.

Default:

```yaml
nextcloud_install_dir: /var/www/nextcloud
```

### `nextcloud_data_dir`

Type: `path`. Required: `false`.

Absolute data directory outside the application document root; selected during
initial installation.
Changing this input does not migrate stored data or change an initialized
instance.
The role persists httpd_sys_rw_content_t for this path when SELinux is enabled.

Default:

```yaml
nextcloud_data_dir: /srv/www/nextcloud
```

### `nextcloud_config_dir`

Type: `path`. Required: `false`.

Configuration directory outside the application and data directories, containing
config.php and ansible.config.php.
Configure NEXTCLOUD_CONFIG_DIR with this path in the PHP web runtime; the role
supplies it to occ and cron.
Choose this path before installation; the role does not relocate an initialized
instance.

Default:

```yaml
nextcloud_config_dir: /etc/nextcloud
```

### `nextcloud_user`

Type: `str`. Required: `false`.

Existing PHP runtime account used for application ownership, occ and background
jobs.
Empty selects www-data on Debian and Ubuntu, apache on AlmaLinux and Fedora, or
wwwrun on openSUSE.

Default:

```yaml
nextcloud_user: ''
```

### `nextcloud_group`

Type: `str`. Required: `false`.

Existing group shared by the PHP runtime and webserver for application file
access.
Empty selects www-data on Debian and Ubuntu, apache on AlmaLinux and Fedora, or
www on openSUSE.

Default:

```yaml
nextcloud_group: ''
```

### `nextcloud_php_binary`

Type: `path`. Required: `false`.

Absolute PHP CLI executable with the same version and required extensions as the
web runtime.

Default:

```yaml
nextcloud_php_binary: /usr/bin/php
```

### `nextcloud_sebooleans`

Type: `dict`. Required: `false`.

Mapping of SELinux boolean names to boolean states, applied persistently when
SELinux is enabled.
The default prohibits unified webroot writes; additional network and storage
permissions require explicit entries.
These settings affect all processes covered by the host HTTP policy; only listed
booleans are managed.
Override the complete mapping for the services in use; an empty mapping leaves
booleans unchanged.

Default:

```yaml
nextcloud_sebooleans:
  httpd_unified: false
```

### `nextcloud_database_type`

Type: `str`. Required: `false`.

Database engine used during initial installation; mysql includes MariaDB.

Default:

```yaml
nextcloud_database_type: mysql
```

### `nextcloud_database_host`

Type: `str`. Required: `false`.

Database host, optionally with a port or socket, used during initial
installation.
Remote connections require a separately secured network or database transport.

Default:

```yaml
nextcloud_database_host: localhost
```

### `nextcloud_database_name`

Type: `str`. Required: `false`.

Prepared empty database used during initial installation.

Default:

```yaml
nextcloud_database_name: nextcloud
```

### `nextcloud_database_user`

Type: `str`. Required: `false`.

Prepared account with full privileges on the Nextcloud database.

Default:

```yaml
nextcloud_database_user: nextcloud
```

### `nextcloud_database_password`

Type: `str`. Required: `true`.

Initial database password supplied through Vault or another secret store.
Must be nonempty without newlines or leading or trailing whitespace; later runs
do not rotate it.

### `nextcloud_admin_user`

Type: `str`. Required: `false`.

Administrator account created during initial installation; later runs do not
rename it.

Default:

```yaml
nextcloud_admin_user: admin
```

### `nextcloud_admin_password`

Type: `str`. Required: `true`.

Initial administrator password supplied through Vault or another secret store.
Must be nonempty without newlines or leading or trailing whitespace; later runs
do not reset it.

### `nextcloud_trusted_domains`

Type: `list`. Required: `true`.

Nonempty list of accepted hostnames or IP addresses, optionally with ports,
without wildcards or URL paths.
The managed configuration replaces the complete trusted_domains list.

### `nextcloud_url`

Type: `str`. Required: `true`.

Public HTTPS base URL used for links generated by background jobs; configure TLS
separately.

### `nextcloud_config`

Type: `dict`. Required: `false`.

Additional Nextcloud system settings using JSON-compatible values, including
Redis, mail and proxy settings.
Bootstrap identity, database connection, data directory, trusted_domains and
overwrite.cli.url keys are reserved.
Values are literal data and cannot contain executable PHP; output and diffs are
suppressed.

Default:

```yaml
nextcloud_config:
  debug: false
```

### `nextcloud_apps`

Type: `list`. Required: `false`.

Unique App Store identifiers to install and keep enabled, for example calendar
or contacts.
Missing apps are installed, disabled apps are enabled and enabled apps remain
unchanged.
Unlisted apps are preserved; removing an entry does not uninstall it. App
upgrades remain separately managed.

Default:

```yaml
nextcloud_apps: []
```

## Managed Files

- `/var/cache/nextcloud/latest.tar.bz2` Verified upstream archive retained in a
  root-only cache; downloaded only when the application entry point is absent.
- `<nextcloud_config_dir>/ansible.config.php` Managed PHP configuration
  fragment, validated with PHP CLI, mode 0640; configuration changes have
  module-provided backups.
- `<nextcloud_config_dir>/config.php` Created and maintained by Nextcloud; the
  role enforces mode 0640 without replacing its content.
- `/etc/cron.d/nextcloud` Executes cron.php every five minutes as
  nextcloud_user.

## Check Mode

Check mode predicts supported file and package changes without installing or
initializing Nextcloud.

- Before the first installation, archive download, extraction and application
  configuration are skipped because occ is unavailable.
- On installed instances, read-only occ commands inspect state; initialization
  and background mode commands do not run in check mode.

## Service Behavior

Enable and start cron after configuration. PHP and webserver services remain
under their owning configuration. Nextcloud loads the managed PHP fragment when
handling requests; there are no service restart handlers.

## Security Notes

- Installation passwords must be nonempty single lines without leading or
  trailing whitespace because the installer trims input.
- Application files are readable by the runtime group and inaccessible to other
  users. Configuration files use mode 0640 and the data directory uses mode
  0750. Additional configuration, including secrets, is readable by the PHP
  runtime group.
- The default data directory is /srv/www/nextcloud, separate from the
  application at /var/www/nextcloud. SELinux-enabled hosts receive persistent
  httpd_sys_rw_content_t labels for writable paths.
- The default nextcloud_sebooleans disables httpd_unified and leaves other
  booleans unchanged. Configure httpd_can_network_connect_db for TCP databases
  and httpd_can_network_connect when external connections are required. These
  permissions affect the host HTTP policy, including other applications; include
  only those needed by your deployment. Optional entries include
  httpd_can_connect_ldap, httpd_can_sendmail, httpd_use_cifs, httpd_use_nfs and
  httpd_use_fusefs. With unrestricted outbound connections disabled, Redis may
  need httpd_can_network_memcache. Preserve httpd_unified=false in overrides.
  Application files receive a read-only content label; write labels cover apps,
  configuration, data, .htaccess, .user.ini and AWS SDK logs.
- Configuration defaults to /etc/nextcloud outside the webroot. PHP must retain
  access to /dev/urandom, including when open_basedir is used.
- Configure HSTS with at least 15552000 seconds. Disable HTTP compression and
  enforce Nextcloud's security headers on static responses too.
- Restrict administration with allowed_admin_ranges in nextcloud_config when the
  permitted networks are known. High-security deployments can set
  enable_previews to false.

## Operational Notes

- Enable OPcache, preserve comments with opcache.save_comments=1 and retain
  timestamp validation. The tested revalidation interval is two seconds.
  Disabling revalidation can leave changed configuration stale until PHP-FPM
  restarts.
- For production caching, provision APCu and Redis before configuring
  memcache.local, memcache.locking and redis in nextcloud_config. Enable
  apc.enable_cli when CLI jobs use APCu. Redis locking reduces database
  contention; this role does not provision the cache service.
- The latest release is selected only on initial deployment. An existing occ
  prevents archive replacement, so reruns do not upgrade or downgrade Nextcloud.
  Use the documented Nextcloud upgrade procedure separately; pending database
  upgrades cause the role to fail.
- Database settings, administrator credentials and the data path initialize a
  new instance; they do not migrate or reset an installed instance. Keep
  installation paths and runtime ownership stable after provisioning.
- Additional settings live in a separate configuration fragment. Removing a
  setting from nextcloud_config removes its managed override; an underlying
  config.php value may then apply.
- nextcloud_apps installs missing apps and enables disabled ones without forcing
  compatibility or replacing installed versions. Unlisted apps are preserved.
  Removing an entry does not remove the app or its data. Manage app upgrades
  separately.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| Debian | Debian | latest | [jomrr/molecule-debian:latest](https://hub.docker.com/r/jomrr/molecule-debian) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Leap | latest | [jomrr/molecule-opensuse-leap:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-leap) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |
| Debian | Ubuntu | latest | [jomrr/molecule-ubuntu:latest](https://hub.docker.com/r/jomrr/molecule-ubuntu) |

## Example Playbook

### Install on a prepared MariaDB and PHP host

Store both installation passwords in Ansible Vault.

```yaml
---
- name: Install Nextcloud
  hosts: nextcloud
  gather_facts: true
  roles:
    - role: jomrr.nextcloud
      nextcloud_trusted_domains:
        - cloud.example.org
      nextcloud_url: https://cloud.example.org
      nextcloud_database_password: "{{ vault_nextcloud_database_password }}"
      nextcloud_admin_password: "{{ vault_nextcloud_admin_password }}"
      nextcloud_config:
        default_phone_region: DE
        maintenance_window_start: 1
        debug: false
      nextcloud_apps:
        - calendar
        - contacts
```

## References

- [Nextcloud installation manual](https://docs.nextcloud.com/server/latest/admin_manual/installation/index.html)
- [Nextcloud command-line installation](https://docs.nextcloud.com/server/latest/admin_manual/installation/command_line_installation.html)
- [Nextcloud system requirements](https://docs.nextcloud.com/server/latest/admin_manual/installation/system_requirements.html)
- [Nextcloud PHP configuration](https://docs.nextcloud.com/server/latest/admin_manual/installation/php_configuration.html)
- [Nextcloud server tuning](https://docs.nextcloud.com/server/latest/admin_manual/installation/server_tuning.html)
- [Nextcloud background jobs](https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/background_jobs_configuration.html)
- [Nextcloud SELinux configuration](https://docs.nextcloud.com/server/latest/admin_manual/installation/selinux_configuration.html)
- [Nextcloud app commands](https://docs.nextcloud.com/server/latest/admin_manual/occ_apps.html)
- [Nextcloud hardening guidance](https://docs.nextcloud.com/server/latest/admin_manual/installation/harden_server.html)
- [Apache FastCGI environment handling](https://httpd.apache.org/docs/2.4/mod/mod_proxy_fcgi.html#proxyfcgisetenvif)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
