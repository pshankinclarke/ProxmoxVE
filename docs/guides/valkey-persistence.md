# Valkey persistence and Proxmox backups

Persistence in Valkey refers to retaining data across restarts by writing it to disk. Proxmox provides backup and restore for LXC containers, so once Valkey is persisting to disk inside the container, Proxmox can back it up and restore it as needed.

By default, Valkey includes RDB defaults: `3600 1 300 100 60 10000`, but depending on your use case, you might want to change this. Below are some common persistence profiles and an example of how they behave with Proxmox backups.

## Persistence profiles

> All example snippets can be placed in `/etc/valkey/valkey.conf` and then restart the service:
```bash
sudo systemctl restart valkey
```

1.) RDB (default persistence):
     This uses point-in-time snapshots (`dump.rdb`) to persist data. On a fresh Debian-based Valkey LXC, RDB is enabled with the following defaults ( retrivable via `valkey-cli CONFIG GET save` ):
     ```conf
     save 3600 1 300 100 60 10000
     appendonly no
     ```

2.) AOF (Append Only File) persistence:
    Debian does not enable AOF by default. Once enabled, AOF logs every write operation by the server. On restart, Valkey will replay the log to rebuild your dataset.
    Example configuration:
    ```conf
    appendonly yes
    save ""  # disable RDB snapshots
    ```

3.) No persistence:
    This disables persistence entirely so the instance behaves much more like a cache. All data is lost on restart.
    Example configuration:
    ```conf
    save ""
    appendonly no # already the default
    ```

4.) RDB + AOF hybrid:
    If you want something like what PostgreSQL offers, enable both RDB and AOF.
    Example configuration:
    ```conf
    save 3600 1 300 100 60 10000 # RDB defaults
    appendonly yes
    ```

> In all cases, Debian's default paths for data files remain unchanged, so RDB and AOF files continue to live under `/var/lib/valkey` where Proxmox CT backups can find them.

## Proxmox backup integration steps

1.) Edit `/etc/valkey/valkey.conf` inside the container and set one of the profiles, then restart: 
    You should see files like this, depending on the profile:
    ```bash
    /var/lib/valkey:
    appendonlydir  dump.rdb

    /var/lib/valkey/appendonlydir:
    appendonly.aof.1.base.rdb
    appendonly.aof.1.incr.aof
    appendonly.aof.manifest
    ```

2.) Write some data to Valkey: 
    ```bash
    PASS="$(cat ~/valkey.creds)"

    valkey-cli -a $PASS SET foo bar
    valkey-cli -a $PASS SET counter 42
    ```

3.) On the Proxmox host, run a backup of the container:
    ```bash
    vzdump <CTID> --mode snapshot --storage <STORAGE> --compress zstd
    ```
    This creates an archive at a templatized path: 
    ```text
    /var/lib/vz/dump/vzdump-lxc-<CTID>-YYYY_MM_DD-HH_MM_SS.tar.zst
    ``` 

4.) Restore into a new container:
    ```bash
    pct restore <NEW_CTID> /var/lib/vz/dump/vzdump-lxc-<CTID>-YYYY_MM_DD-HH_MM_SS.tar.zst --storage <STORAGE>
    pct start <NEW_CTID>
    pct enter <NEW_CTID>
    ```

5.) Verify the data:
    ```bash
    PASS="$(cat ~/valkey.creds)"

    valkey-cli -a $PASS GET foo
    valkey-cli -a $PASS GET counter
   ```

If you get `bar` and `42`, then your persistence and backup are successfully capturing and restoring your Valkey dataset!
