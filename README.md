 Ansible Role: restic
=======================

> **Beta:** This role is in beta status.

[![🎭 Tests](https://github.com/roles-ansible/ansible_role_restic/actions/workflows/test.yml/badge.svg)](https://github.com/roles-ansible/ansible_role_restic/actions/workflows/test.yml)
[![license](https://raw.githubusercontent.com/roles-ansible/ansible_role_restic/main/.github/license.svg)](https://github.com/roles-ansible/ansible_role_restic/blob/main/LICENSE)
[![Ansible Galaxy](https://raw.githubusercontent.com/roles-ansible/ansible_role_restic/main/.github/galaxy.svg)](https://galaxy.ansible.com/do1jlr/restic)

 Description
-------------
[Restic](https://github.com/restic/restic) is a versatile Go based backup
solution which supports multiple backends, deduplication and incremental
backups.

This role installs restic on a client, configures the backup repositories
and optionally sets systemd timer or cronjobs to run the backups.
Aditionally, it will setup executable scripts to run a Backup manually.

> This Project borrowed heavily from the
> [donat-b/ansible-restic](https://github.com/donat-b/ansible-restic) and
> the [https://github.com/arillso/ansible.restic](https://github.com/arillso/ansible.restic)
> ansible role. We try to make this role more easy to anderstand and modern by using systemd timer,
> /etc/crontab to define backup paths, more absolute paths and less options. (no S3 Storage, No Windows...)

### Backup Scripts
This role will create a backup script and a file with credentials usable with the `source` command on linux for each backup in the `restic_script_dir`.
These executable scripts can be used to manually trigger a backup action, but
are also used for automated backups if you have set `restic_create_schedule` variable to true.
Make sure to not change the files manually, as this can interfere with your
backups quite a bit.

on Linux, if you want to take a manual snapshot, you can run the backup like this:
```bash
$ /path/to/backup/script/backup-example.sh
```
by default, such a snapshot will be given the tag `manual`, so you can distinguish
them from automatically created snapshots. You can also append more tags by
simply appending them:
```bash
$ /path/to/backup/script/backup-example.sh --tag deployment
```

### CRON / Scheduled Tasks
In order to make use of defined backups, they can be automatically setup as
scheduled tasks. You have to be aware of the fact that (on linux systems at
least) you need to have administrator permissions for configuring such an action.

If you cannot use the automatic creation of the tasks, you can still make use
of the generated scripts. If you are for example on a shared hosting server
and can define a cronjob via a webinterface, simply add each backup file to
be executed. Make sure to prefix the command with `CRON=true` to imply that the
snapshot was created via a scheduled task:
```bash
CRON=true /path/to/backup/script/backup-example.sh
```
## Installation

```bash
ansible-galaxy install arillso.restic
```
## Requirements
* bzip2
## Role Variables

| Name                   | Default                             | Description                                                                 |
| ---------------------- | ----------------------------------- | --------------------------------------------------------------------------- |
| `restic_url`           | `undefined`                         | The URL to download restic from. Use this variable to overwrite the default |
| `restic_version`       | `'0.12.1'`                          | The version of Restic to install                                            |
| `restic_download_path` | `'/opt/restic'`                     | Download location for the restic binary                                     |
| `restic_install_path`  | `'/usr/local/bin'`                  | Install location for the restic binary                                      |
| `restic_script_dir`    | `'/opt/restic'`                        | Location of the generated backup scripts                                 |
| `restic_log_dir`       | `'{{ restic_script_dir }}/log'`     | Location of the logs of the backup scripts                                  |
| `restic_repos`         | `{}`                                | A dictionary of repositories where snapshots are stored. *(More Info: [Repos](#Repos))* |
| `restic_backups`       | `{}` (or `[]`)                      | A list of dictionaries specifying the files and directories to be backed up *(More Infos: [Backups](#Backups))* |
|`restic_create_schedule`| `false`                             | Should we schedule each backup. Either via cronjob or via systemd timer.|
| `restic_schedule_type` | `systemd`                           | Here you can define if we create a ``cronjob`` or a ``systemd`` timer. If it fails to create a systemd timer, a cronjob will be created. |
| `restic_dir_owner`     | `'{{ansible_user}}'`                | The owner of all created dirs                                               |
| `restic_dir_group`     | `'{{ansible_user}}'`                | The group of all created dirs                                               |
| `restic_no_log`        | `true`                              | set to false to see hidden ansible logs                                     |
| `restic_do_not_cleanup_cron ` | `false`                      | we changed the cron location and clean up the old one. You can skip the cleanup here |
| `restic__cache_config` | `false`                             | configure custom cache directory                                            |
| `restic__cache_dir`    | `'~/.cache/restic'`                 | define custom cache directory                                               |

### Repos
Restic stores data in repositories. You have to specify at least one repository
to be able to use this role. A repository can be local or remote (see the
official [documentation](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html)).

> **Using an SFTP repository**
>
> For using an SFTP backend, the user needs passwordless access to the host.
> Please make sure to distribute ssh keys accordingly, as this is outside of
> the scope of this role.

Available variables:

| Name                    | Required | Description                                                                                                                                                                                                                                                                                                             |
| ----------------------- |:--------:| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `location`              |   yes    | The location of the Backend. Currently, [Local](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#local), [SFTP](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#sftp), [S3](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#amazon-s3) and [B2](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#backblaze-b2) are supported |
| `password`              |   yes    | The password used to secure this repository                                                                                                                                                                                                                                                                             |
| `init`                  |    no    | Describes if the repository should be initialized or not. Use `false` if you are backuping to an already existing repo.                                                                                                                                                                                                 |

Example:
```yaml
restic_repos:
  local:
    location: /srv/restic-repo
    password: securepassword1
    init: true
  remote:
    location: sftp:user@host:/srv/restic-repo
    password: securepassword2
    init: true
```

### Backups
A backup specifies a directory or file to be backuped. A backup is written to a
Repository defined in `restic_repos`.

Available variables:

| Name               |      Required (Default)       | Description  |
| ------------------ |:-----------------------------:| ------------ |
| `name`             |              yes              | The name of this backup. Used together with pruning and scheduling and needs to be unique. |
| `repo`             |              yes              | The name of the repository to backup to. |
| `src`              | yes (unless `stdin` == `true`) | The source directory or file |
| `stdin`            |              no               | Is this backup created from a [stdin](https://restic.readthedocs.io/en/stable/040_backup.html#reading-data-from-stdin)? |
| `stdin_cmd`        | no (yes if `stdin` == `true`) | The command to produce the stdin. |
| `stdin_filename`   |              no               | The filename used in the repository. |
| `tags`             |              no               | Array of default tags  |
| `keep_last`        |              no               | If set, only keeps the last n snapshots.  |
| `keep_hourly`      |              no               | If set, only keeps the last n hourly snapshots.                                                                                                                              |
| `keep_daily`       |              no               | If set, only keeps the last n daily snapshots.                                                                                                                               |
| `keep_weekly `     |              no               | If set, only keeps the last n weekly snapshots.                                                                                                                              |
| `keep_monthly`     |              no               | If set, only keeps the last n monthly snapshots.                                                                                                                             |
| `keep_yearly `     |              no               | If set, only keeps the last n yearly snapshots.                                                                                                                              |
| `keep_within`      |              no               | If set, only keeps snapshots in this time period.                                                                                                                            |
| `keep_tag`         |              no               | If set, keep snapshots with this tags. Make sure to specify a list.                                                                                                          |
| `prune`            |         no (`false`)          | If `true`, the `restic forget` command in the script has the [`--prune` option](https://restic.readthedocs.io/en/stable/060_forget.html#removing-backup-snapshots) appended. |
| `scheduled`        |         no (`false`)          | If `restic_create_schedule` is set to `true`, this backup is scheduled and tries to create a systemd timer unit. If it fails, it is creating a cronjob. |
| `schedule_oncalendar` |  ``'*-*-* 02:00:00'``      | The time for the systemd timer. Please notice the randomDelaySec option. By Default the backup is done every night at 2 am (+0-4h). But only if scheduled is true.  |
| `schedule_minute`  |           no (`*`)            | Minute when the job is run. ( 0-59, *, */2, etc ) |
| `schedule_hour`    |           no (`2`)            | Hour when the job is run. ( 0-23, *, */2, etc )  |
| `schedule_weekday` |           no (`*`)            | Weekday when the job is run.  ( 0-6 for Sunday-Saturday, *, etc ) |
| `schedule_month`   |           no (`*`)            | Month when the job is run. ( 1-12, *, */2, etc )  |
| `exclude`          |           no (`{}`)           | Allows you to specify files to exclude. See [Exclude](#exclude) for reference. |
| `disable_logging`  |           no                  | Optionally disable logging  |
| `log_to_journald`  |           no                  | Optionally switch logging to journald with the name of the backup job as the tag |
| `mail_on_error`    |           no                  | Optionally send a mail if the backupjob will fail *(mailx is required)* |
| `mail_address`     |  if `mail_on_error` is true   | The mail addressto recive mails if you enabled ``mail_on_error``. |

Example:
```yaml
restic_backups:
  data:
    name: data
    repo: remove
    src: /path/to/data
    scheduled: true
    schedule_oncalendar: '*-*-* 01:00:00'
  database:
    name: database
    repo: remove
    stdin: true
    stdin_cmd: pg_dump -Ubackup db_name
    stdin_filename: db_name_dump.sql
    scheduled: true
    schedule_oncalendar: '*-*-* 01:30:00'
```

> You can also specify restic_backups as an array, which is a legacy feature and
> might be deprecated in the future. currently, the name key is used for
> namint the access and backup files

#### Exclude
the `exclude` key on a backup allows you to specify multiple files to exclude or
files to look for filenames to be excluded. You can specify the following keys:
```yaml
exclude:
    exclude_cache: true
    exclude:
        - /path/to/file
    iexclude:
        - /path/to/file
    exclude_file:
        - /path/to/file
    exclude_if_present:
        - /path/to/file
```
Please refer to the use of the specific keys to the
[documentation](https://restic.readthedocs.io/en/latest/040_backup.html#excluding-files).

## Dependencies
none
## Example Playbook

```yml
- hosts: all
  roles:
    - restic
```

## License

This project is under the MIT License. See the [LICENSE](LICENSE) file for the full license text.
