# SOC Infra Docker — Quick start

## Overview

This repository contains a `docker-compose.yml` and minimal config files to run a small SOC stack locally (Elasticsearch, Cassandra, Redis, MySQL/Postgres DB, Cortex, MISP, Kibana, TheHive, etc.).

The instructions below reproduce the sequence of shell commands you followed — written as a step-by-step guide so you can reproduce the setup.

> **Security note:** keep your secrets (play.http.secret.key, Cortex API keys, DB passwords) private and never commit them to a public repo.

---

## Prerequisites

* Docker & Docker Compose installed (compatible versions).
  
For Docker installation, follow Steps 1 & 2: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04

For Docker Compose: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-20-04

* Verify with:

```bash
docker --version
docker-compose --version
```

* `openssl` available to generate secrets.
* Sane host resources (Elasticsearch and Cassandra need memory/CPU).

---

## Files & directories created in the process

The commands below create the folders and empty config files required by the `docker-compose.yml`:

```
mkdir soc
sudo nano docker-compose.yml   # (create and paste your compose file)

# persistent volumes and app folders
mkdir -p ./vol/elasticsearch_data ./vol/elasticsearch_logs \
         ./vol/cassandra-data ./vol/mysql \
         ./vol/data ./vol/index \
         ./thehive ./cortex

# empty app config placeholders
touch ./thehive/application.conf ./cortex/application.conf
```

---

## Ownership & permissions — why and how

Set the folder owner to your current user UID/GID so containers that write files to the host do not create root-owned files.

```bash
sudo chown -R $(id -u):$(id -g) ./vol ./thehive ./cortex
```

Set permissions:

* `chmod -R 755 ./vol ./thehive ./cortex` — safer: world-readable, owner writable (recommended for production).
* `chmod -R 775 ./vol ./thehive ./cortex` — friendlier for testing when multiple local users/groups may need write access.

Use the minimum permission that works for your environment.

---

## Generate TheHive and Cortex secret key

TheHive & Cortex needs a `play.http.secret.key` on its `application.conf` (this is an example of how you created it):

```bash
openssl rand -hex 32
# copy the output (do not commit it)
```

Open `./thehive/application.conf` and paste the secret in the correct place.

Same thing for `./cortex/application.conf`

---

## Fill the config files

Edit the application files you created earlier:

```bash
sudo nano ./thehive/application.conf
# paste the play.http.secret.key and other TheHive settings

sudo nano ./cortex/application.conf
# configure Cortex settings (DB, storage, etc.)
```


---

## Start the stack (step-by-step)

1. **Start core services EXCEPT TheHive** — start Elasticsearch, Cassandra, Redis, DB, Cortex, MISP, MISP modules, Kibana first:

```bash
sudo docker-compose up -d \
  elasticsearch cassandra redis db cortex misp misp-modules kibana
```

2. **Wait for services to be healthy.**

   * Check running containers:

```bash
sudo docker-compose ps
```

* Follow logs for a specific container (example: Cortex):

```bash
sudo docker logs -f cortex
```

* For Elasticsearch, wait for the cluster to be green/healthy before starting services that depend on it.

3. **Get the Cortex API key**

   * After Cortex is up, obtain the API key from the Cortex admin UI or from the container logs / endpoint your deployment exposes.

   * Copy the Cortex API key (keep it secret).

4. **Paste the Cortex API key into TheHive config**

   Edit TheHive `application.conf` and add the Cortex API key where TheHive expects to reach Cortex (so TheHive can call Cortex analyzers):

```bash
sudo nano ./thehive/application.conf
# add or update the cortex.api key or thehive connector settings with the API key
```

5. **Start TheHive**

```bash
sudo docker-compose up -d thehive
```

6. **Verify end-to-end**

* Ensure TheHive connects to Cortex and MISP and that analyzers are reachable.
* Check logs for errors:

```bash
sudo docker-compose logs -f thehive cortex
```

* Visit the UIs (Kibana, MISP, TheHive, Cortex) as required to confirm functionality.

---

## Example Useful Commands / Quick reference

```bash
# show containers and status
sudo docker-compose ps

# follow combined logs
sudo docker logs -f --tail=200

# follow a single service logs
sudo docker logs -f cortex

# restart a single service
sudo docker-compose restart cortex

# stop everything
sudo docker-compose down

# view recent logs from docker directly
sudo docker logs --tail 200 <container_name_or_id>
```


