---
title: "Low-effort, reliable self-hosting"
date: 2026-05-11
draft: true
ShowToc: true
---

## Goals

| Goal   | Proposed solution   |
| ------ | ----- |
| Easy to maintain | One docker-compose, one Caddyfile, in git. |
| Data safety | Backups with borgmatic (1 config file does everything) | 
| Reasonable availability | Restore the whole server in one `git clone` and one [`borgmatic extract`](https://torsion.org/borgmatic/).    | 
| Reasonable security | Stable distro, ssh with key only, automatic linux security upgrades | 
| Basic monitoring | Healthcheck service that sends emails. | 


## Structure

Single docker-compose. Every service gets its config and data volume (if needed).

The whole data folder will be included in the incremental backups, while compose + configs are versioned in git.

```
├── compose.yaml      # versioned in git 
│   │
│   ├── Service A 
│   └── ...         
│
├── .env secrets      # in password manager
│
├── /data             # backed-up with borg
│   ├── A
│   └── ...
│
└── /config           # versioned in git
    ├── A
    └── ...
```

{{< callout type="info" >}}
Database data still lives in /data but will need to be excluded from incremental backups in favor of [borgmatic's built-in database backup](https://torsion.org/borgmatic/how-to/backup-your-databases/).
{{< /callout >}}



Ports are exposed only to localhost...
```
Service name         Network         Ports   
--------------------------------------------------------
nextcloud            nextcloud       127.0.0.1:8001:8001
nextcloud-db         nextcloud
Caddy                mode=host
```

...and Caddy exposes to the internet and takes care of TLS (`config/Caddyfile`)
```
nextcloud.example.com {
  reverse_proxy 127.0.0.1:8001
}
```


## Security

- Auth by ssh key only.

{{< callout type="info" >}}
Check that no user has a hash in `cat /etc/shadow`

```
/etc/shadow

user1:!:...            # ! means key auth. Good.
user2:$y$j9T$zYHn:...  # Hash. If any user has this, do not deploy!

```
{{< /callout >}}

- [Enable automatic security upgrades](https://wiki.debian.org/PeriodicUpdates) with unattended-upgrades
- Mount volumes as [readonly](https://docs.docker.com/engine/storage/bind-mounts/#options-for---volume) when relevant (`./folder:/var/folder:ro`).
- Use `user: "1000:1000"`. I recommend [linxserver's images](https://www.linuxserver.io/our-images) that always include this.
- Do not mount docker sockets `-v /var/run/docker.sock:/var/run/docker.sock`

### Limit access to your git repo

When you clone your git repository, use a key that only has access to this repo, in case the server is compromised.

With Github :
- create a keypair  
- define a host in ssh-config
```
Host infra
  HostName github.com
  User git
  IdentityFile ~/infra/id_infra
```
- set up a deployment key on github in repo settings w/ public key
- clone with hostname in ssh-config (`git clone git@infra/repo`)

---

## Monitoring

I use a [bash script](#basic-monitoring-script) to report down services to my inbox, as well as excessive load average or low storage.


# Appendix

### Borg config

### Distribution choice
Debian. Stable and predictable with minimal maintenance. Conservative package policy.
