# Moodle LMS - Deployment Guide.


Steps to deploy [Moodle LMS](https://moodle.org) via docker-compose and Kubernetes.
Uses branch [404_STABLE](https://github.com/moodle/moodle/tree/MOODLE_404_STABLE)
Uses Docker base image [moodlehq/moodle-docker](https://github.com/moodlehq/moodle-docker).

## Docker-Compose


1. Update the `.env` file appropriately. 
Please note that we are using v3 compose files which do not use `.env` files by default.
Instead, `docker-compose.yml` sources the `.env` file into the microservices manually.

2. Deploy. 
This builds the image locally. 
```shell
docker-compose up -d
```


## Kubernetes

Requisites (run only once)


### Context

1. Source envfile 

```shell
source ./k8s/overlays/prod/.env
```

2. Create dedicated namespace for Moodle
  and a Kubernetes app instance for customer eg. `fhcgr`.
  These are interpolated by envsubst in the Kustomize templates.

```shell
export \
  NAMESPACE=tess \
  INSTANCE=fhcgr \
  TAG=latest 


kubectl create namespace $NAMESPACE
```

### Database

Moodle install script requires ready database instance on your kubernetes cluster.
in the `database` namespace.

Supported database engines in `externalDatabase.type` are: `mysqli`, `mariadb`, `pgsql`, `auroramysql`
Hints below for MariaDB and PostgreSQL.


### MariaDB Secrets

1. Copy existing MariaDB secrets from the "database" namespace to this project,

```shell
kubectl -n database get secrets/mariadb -o yaml | \
  yq 'del(.metadata.creationTimestamp, .metadata.uid, .metadata.resourceVersion, .metadata.namespace)' | \
  kubectl apply -n $NAMESPACE -f -
```

Nota: We will use above created `secrets/mysql` in place of `mariadb.mariadbRootPassword`.
This will be referenced use by the `externalDatabase.existingSecret` config below.
Subsequently, in chart config, reference existing mysql database resp. to its namespace ie. `mysql.database`.

2. Create the database

```shell

# Forward pgpool service to localhost:5432
kubectl -n database port-forward svc/mariadb 3306:3306

# Connect to MariaDB as user "root"
DB_PASSWORD=$(kubectl -n database  get secrets/mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 -d)
mysql -u root -p$DB_PASSWORD -h localhost

# Then, in MySQL shell, run:
# create database fhcgr_moodle;
```

### PostgreSQL Secrets

1. Obtain `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD` for your existing db instance.

Note that we do NOT use the `DB_PASSWORD` env. In this particular setting, the db root password is pulled manually a Secret resource existing in our production cluster and
injected into the `$NAMESPACE` namespace using a [ConfigMap]( docker-moodle/k8s/overlays/prod/kustomization.yaml). 

Eg., following example extracts existing PostgreSQL db secret from our `database` namespace, and injects it into the moodle namespace.
Nota: our root `db_username` defaults to `postgres`.

```shell
kubectl -n database get secrets/pgpool -o yaml | \
  yq 'del(.metadata.creationTimestamp, .metadata.uid, .metadata.resourceVersion, .metadata.namespace)' | \
  kubectl apply -n $NAMESPACE -f -
```

2. Create database in PostgreSQL

```shell
kubectl -n database port-forward svc/pg-pgpool 5432

# Connect to PostgreSQL as user "root"
DB_PASSWORD=kubectl -n database  get secrets/mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 -d
psql postgresql://$DB_USER:$DB_PASSWORD@localhost:5432 "create database "$DB_NAME" encoding utf8;"
```

### Container image

Build and push the image to my private DockerHub

```shell
docker build ./docker -t ceduth/moodle:$TAG \
  && docker push ceduth/moodle:$TAG
```

### Deploy

Follow up with additional steps for [GKE](./docs//GKE-prep.md) if applies.

This setup uses [Kustomize](https://kustomize.io/) layere-based 
configuration management.

By default deploys and exposes Moodle behind a Nginx ingress controller pre-installed on GKE. Check []() for instructions to setup a Nginx Ingress Controller with `CertManager` that provisions TLS certificates for an entire cluster.

1. Set your `.env` file appropriately in directory `docker-moodle/k8s/`.
Note, we have set `SSL_PROXY=true` in the [env file](./k8s/overlays/prod/) since we seat behind Nginx Ingress proxy.

2. Create Moodle user and email

```shell
kubectl -n $NAMESPACE create secret/fhcgr-moodle  \
  --from-literal=moodle-password='Th3P@ssw0rd!' \
  --from-literal=moodle-email='francois.carvalho.alvarenga@gmail.com' 
```

3. From `moodle-k8s` directory, run:
Nota: This creates an internal service resource, reachable only within the cluster.

```shell
# kubectl apply -n $NAMESPACE -k ./k8s/overlays/prod
~/DevOps/bin/kustomize build ./k8s/overlays/prod | envsubst | kubectl -n $NAMESPACE apply -f -
```

4. Create an ExternalName resource in the default namespace,
that point to the internal moodle service in namespace designated namespace.

```shell
cat <<EOF | envsubst | kubectl apply -n $NAMESPACE -f -
kind: Service
apiVersion: v1
metadata:
  name: fhcgr-moodle
spec:
  type: ExternalName
  externalName: moodle.$NAMESPACE
EOF
```

5. Finally merge following rule that routes https traffic
into existing Nginx Ingress configuration:

```yaml
---
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: web-ingress
  annotations:
    kubernetes.io/ingress-class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod

...

# <THE-UPDATE>
spec:
  tls:
  ...
  rules:
  - host: "fc.echq.online"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: moodle
            port:
              number: 80
# </THE-UPDATE>

  ```

6. Point your browser at: `https://<MOODLE_URL>`
Note: `MOODLE_URL` was previously set in our [env file](./k8s/overlays/prod/).


### Optional steps

* Fix moodle version in database (did'nt work for me!)

```shell
# Get version from the git repo at
# Or copy version from running pod via ssh at /var/www/html/version.php
# Or fetch it from https://moodledev.io/general/releases
# Cf. https://stackoverflow.com/a/32208453/4615806
# eg. 2022041912.00 for MOODLE_404_STABLE
# --
# Worth reading, but did'nt work for me!
# https://docs.moodle.org/404/en/Installation_FAQ#"Could_not_find_a_top_level_course"
INSERT INTO mdl_config (name, value) VALUES ('version', '2022041912');
```

* Run the installer manually

Refs:
[here](https://docs.moodle.org/404/en/Installing_Moodle#Command_line_installer) and
[there](https://docs.moodle.org/404/en/Administration_via_command_line)

Follow the wizard.
Choose `/var/moodledata` when asked for Moodle data dir (cf. Dockerfile)

```shell
su www-data -c 'php admin/cli/install.php --agree-license' -s /bin/bash
su www-data -c 'php admin/cli/install_database.php --agree-license --adminpass=Th3P@ssw0rd!' -s /bin/bash
```

* Check php info

```shell
# Create info.php in public dir, then
# Open https://fc.echq.online/info.php in brnowser
cat << EOF > /var/www/html/info.php
<?php
    phpinfo();
?>
EOF
```


### Cleanup

```shell
kubectl -n tess delete deployment/fhcgr-moodle
kubectl -n tess delete pvc/fhcgr-moodle-pvc
```





# Resources

- [kustomize-with-multiple-envs ](https://github.com/L-U-C-K-Y/kustomize-with-multiple-envs/tree/main)
# https://github.com/kubernetes-sigs/kustomize/blob/master/examples/secretGeneratorPlugin.md
# https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kustomize



