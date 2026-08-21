---
title: "Recipe: docker + crontab"
date: 2026-05-11
ShowToc: true
---

## Prerequisite: if you start from a base image, is it based on alpine or debian?

The syntax is not the same because debian uses `cron` [^1] while alpine uses `crond` [^2].

Check with:

```
$ docker run --rm ghcr.io/borgmatic-collective/borgmatic:latest cat /etc/os-release

Output:
NAME="Alpine Linux"
```


## 1. From a fresh Debian/Ubuntu base (`crontab`)


**Dockerfile**
```Dockerfile
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y cron && rm -rf /var/lib/apt/lists/*

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

**entrypoint.sh**
```bash
#!/bin/sh
set -e

(printenv | grep -v "^_="; echo "0 2 * * * /my/script.sh >> /var/log/cron.log 2>&1") | crontab -

exec cron -f
```

Unlike with alpine, the environment variables are not available to the script if you miss the `printenv` part [^4]


## 2. From an existing image, based on Alpine: example of borgmatic (`crond`)

**compose.yaml**
```yaml
services:
  borgmatic:
    build: ./borgmatic
```

**borgmatic/Dockerfile**
```Dockerfile
FROM ghcr.io/borgmatic-collective/borgmatic:latest

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"] 
CMD []
```

**borgmatic/entrypoint.sh**
```bash
#!/bin/sh
set -e

borgmatic init ... # Because we override the entrypoint, we need to insert the commands that it would have ran

# 2 AM
echo "0 2 * * * borgmatic 2>&1 | tee -a /var/log/cron.log" > /etc/crontabs/root

exec crond -f
```

Unlike with Debian, environment variables are inherited. No tricks needed [^5]

## Troubleshooting

Does the script work inside the container? 

- Trigger the script manually: `docker exec <container> /my/script.sh`

Does the cron trigger?

- Check if the cron is firing: run it every minute using `* * * * *` and add a log: `echo "$(date) cron fired" >> /var/log/cron.log; /my/script.sh`


{{< callout type="warning" >}}
Note: output goes to /var/log/cron.log only, and will not show up in `docker logs <container>`.
Instead use `docker exec <container> tail -n 500 -f /var/log/cron.log`
{{< /callout >}}


[^1]: https://wiki.debian.org/cron
[^2]: https://wiki.alpinelinux.org/wiki/Cron
[^4]: https://gist.github.com/Yann-P/43b8775c3660474867d5bc2a8b9250f3
[^5]: https://gist.github.com/Yann-P/a819051856a3cb647fd7b1c10b7ed341
