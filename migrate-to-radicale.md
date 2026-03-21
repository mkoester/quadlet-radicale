# About

In case you want to migrate existing calendar and/or contact data from another DAV product.

# How to

1. install vdirsyncer
2. create a directory for the config and syncronization data
3. cd into that directory
4. create a config `vdirsyncer.ini` in that directory (example see below)
5. export VDIRSYNCER_CONFIG=`pwd`/vdirsyncer.ini
6. start `discovery` (confirm to create the collections you want to synchronize), then `sync`


## example config `vdirsyncer.ini`

I migrated from Nextcloud, adjust to your setup (especially `url`, `username`, `password`).

```ini
[general]
status_path = "./vdirsyncer-status/"

# ============================================================
# KALENDER: <Source> → local
# ============================================================

[pair cal_source_to_local]
a = "source_cal"
b = "local_cal"
collections = ["from a"]

[storage source_cal]
type = "caldav"
url = "https://nextcloud.domain.tld/remote.php/dav/calendars/userA/"
username = "userA"
password = "passwordA"

[storage local_cal]
type = "filesystem"
path = "./source-export/calendars/"
fileext = ".ics"

# ============================================================
# KALENDER: local → Radicale
# ============================================================

[pair cal_local_to_radicale]
a = "local_cal"
b = "radicale_cal"
collections = ["from a"]

[storage radicale_cal]
type = "caldav"
url = "https://radicale.domain.tld/userB/"
username = "userB"
password = "passwordB"

# ============================================================
# KONTAKTE: <Source> → local
# ============================================================

[pair contacts_source_to_local]
a = "source_contacts"
b = "local_contacts"
collections = ["from a"]

[storage source_contacts]
type = "carddav"
url = "https://nextcloud.domain.tld/remote.php/dav/addressbooks/userA/"
username = "userA"
password = "passwordA"

[storage local_contacts]
type = "filesystem"
path = "./source-export/contacts/"
fileext = ".vcf"

# ============================================================
# KONTAKTE: local → Radicale
# ============================================================

[pair contacts_local_to_radicale]
a = "local_contacts"
b = "radicale_contacts"
collections = ["from a"]

[storage radicale_contacts]
type = "carddav"
url = "https://radicale.domain.tld/userB/"
username = "userB"
password = "passwordB"

```

## Discovery & Sync

### source to local first

```sh
vdirsyncer discover cal_source_to_local
vdirsyncer discover contacts_source_to_local

vdirsyncer sync cal_source_to_local
vdirsyncer sync contacts_source_to_local
```

### local to radicale next

```sh
vdirsyncer discover cal_local_to_radicale
vdirsyncer discover contacts_local_to_radicale


vdirsyncer sync cal_local_to_radicale
vdirsyncer sync contacts_local_to_radicale
```
