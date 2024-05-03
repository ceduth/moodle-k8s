
# Helm Chart


Based on Bitnami Helm Chart [here](https://bitnami.com/stack/moodle/helm)

Refs:
* [README](https://github.com/bitnami/charts/blob/main/bitnami/moodle/README.md)
* [installing-the-chart](https://github.com/bitnami/charts/tree/main/bitnami/moodle/#installing-the-chart)
* [Container](https://github.com/bitnami/containers/tree/main/bitnami/moodle)


```bash
helm install my-release oci://registry-1.docker.io/bitnamicharts/moodle
```


## Requisites

0. Test the image ;)
Didn't work for me (moodle:4.4, BRANC_404) as of April 30, 2024;
so I'm resorting for now to building the image from official Moodle git repo.

```shell

# Create
docker run -d --name moodle \
  -p 8080:8080 -p 8443:8443 \
  --env ALLOW_EMPTY_PASSWORD=no \
  --env MOODLE_DATABASE_USER=postgres \
  --env MOODLE_DATABASE_PASSWORD='<YOUR-PGPASSWORD>' \
  --env MOODLE_DATABASE_NAME=bitnami_moodle \
  --env MOODLE_DATABASE_TYPE=pgsql \
  --env MOODLE_DATABASE_HOST=192.168.1.10 \
  --volume moodle_data:/bitnami/moodle \
  --volume moodledata_data:/bitnami/moodledata \
  ceduth/moodle:4.4 \
&& docker logs -f moodle

# Test PostgreSQL connection on postgres container
docker exec -it moodle bash
psql -U postgres -h 192.168.1.10 -W bitnami_moodle

# Cleanup
docker stop moodle && docker rm moodle && docker volume prune -a
```

* Chart config

- Set `mariadb.enabled=false` as we're using an external MariaDB server.
- Supported database engines in `externalDatabase.type` are: `mysqli`, `mariadb`, `pgsql`, `auroramysql`

Manually mend the chart's values.yaml as given below.

```yaml

# Database
# ---------
mariadb.enabled: false
allowEmptyPassword: false
externalDatabase.user: "root"
externalDatabase.host: "mariadb.database"
externalDatabase.type: "mariadb"
externalDatabase.database: "fhcgr_moodle"
externalDatabase.existingSecret: "mariadb"

# Network
# --------
# Allocate dedicated (public) IP from OpenLB pool.
metrics.service.type=LoadBalancer

# Auth
# -----
moodleUsername=admin
moodlePassword=Th3P@ssw0rd!
moodleEmail=francois.carvalho.alvarenga@gmail.com

# Uncomment according to needs
# image.debug=true
```
