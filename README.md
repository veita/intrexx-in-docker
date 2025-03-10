# Intrexx in Docker

__ATTENTION:__ Please refer to the Docker Hub page, for a list of available image names.

Intrexx in Docker creates an ensemble of containers to host an Intrexx instance plus a portal. It builds on the basic idea that the portal is not part of the image. Instead, the image is just an Intrexx runtime (stripped heavily from a default intrexx installation) plus the blank portal template. Docker Compose is used as a deployment tool.

There are four use-cases:

1. If the deployment is started for the first time, a blank portal is created.
2. Alternatively, a zip file containing a portal export may be provided. That portal is then imported, instead of creating a blank portal.
3. If the portal volume contains a portal of the current version, it is started.
4. If the portal is not yet of the current version, it is patched and then started.

# Prerequisites

In order to deploy Intrexx as a Docker container, we use [docker-compose](https://docs.docker.com/compose/). This is done, because Intrexx can not be run as a standalone container but needs a database and a search server to work correctly. Optionally, an Nginx server can be useful as reverse proxy. Managing the deployment as a whole is enabled by Docker Compose.

In order to follow these guidelines, you need to have Docker and Docker Compose configured on your machine. Please follow the official installation guidelines:

- [Docker installation](https://docs.docker.com/get-docker/)
- [Docker Compose installation](https://docs.docker.com/compose/install/)

# Usage
First clone this repository anywhere on your machine:
```
git clone https://github.com/UnitedPlanet/intrexx-in-docker
```
Then create an .env file in the directory where the docker-compose.yml is located:
```
cp .env.example .env
```
Optionally, user-defined settings can now be made by adjusting the values of the variables in the .env file accordingly. This includes e.g. the versions of services, the binding of the container ports to the host, the definition of the portal name, the database user and password, etc.

## Create new (blank) portal

This is the most simple use-case. No changes are needed. Just run

```bash
docker compose up -d
```

This will create three containers: database, Solr server and portal. The portal directory will be created in a named Docker volume. This may be used to backup data but also persists the portal between upgrades.

## Create new portal from export

Many users might already have a portal (export) that they wish to deploy in Docker. To achieve that, the portal must be provided in zip format.

As described above, the portal directory (/opt/intrexx/org) is a named volume in the Docker Compose deployment. If a portal zip is provided within this volume, it is used as template for portal creation, instead of the blank portal. This workflow however requires for the volume to be a mounted directory from the host system instead.

In order to make this work at least the two variables `IX_PORTAL_ZIP_NAME` and `IX_PORTAL_DIR_HOST` must be specified via the .env file. In addition, the docker-compose.yml must be adapted as follows:

```yml
# Case 1 (portal is provided from mounted host directory)

intrexx:
  ...
    volumes:
#      - portal-data:/opt/intrexx/org
      - ${IX_PORTAL_DIR_HOST}:/opt/intrexx/org
#      - ${IX_PORTAL_DIR_HOST}:${IX_PORTAL_ZIP_MNTPT}
```

If you don't want to bind-mount /opt/intrexx/org to `IX_PORTAL_DIR_HOST` on the host and /opt/intrexx/org in the container should still be within a named volume, you can use an additional volume containing only the portal zip. In this case, the (absolute) path to mount point within the container must be provided via the .env file (`IX_PORTAL_ZIP_MNTPT`). The docker-compose.yml must then be adjusted as follows:

```yml
# Case 2 (portal is provided from additional volume)

intrexx:
  ...
    volumes:
      - portal-data:/opt/intrexx/org
#      - ${IX_PORTAL_DIR_HOST}:/opt/intrexx/org
      - ${IX_PORTAL_DIR_HOST}:${IX_PORTAL_ZIP_MNTPT}

```

In both cases, the portal can again be started by running

```bash
docker compose up -d
```

## Destroy deployment

To completely destroy a deployment, use

```bash
docker compose down -v
```
__Attention:__ This will also remove the Docker volumes so no data is left! To stop the volumes without removing them (so they can be restarted), use

```bash
docker compose down
```

## Upgrade deployment

To update to a newer version of Intrexx, optionally configure the desired version in the .env file or leave as is, to use the latest available version. Then run these commands:

```bash
docker compose down
docker compose pull
docker compose up
```

By doing so, the Intrexx container will be destroyed, the image updated and started again as a new Intrexx container. Meanwhile the volume persists. On startup the already present portal is detected, patched to the new Intrexx version and started.
Please be aware that the startup may take extended time because of the required portal patch.

## Configure Nginx as reverse proxy

If you want to use Nginx as a frontend webserver and thereby enable SSL encryption, the procedure depends on whether the portal is directly mounted from the host machine or contained in an extra volume.

In both cases you follow these steps:

- In the `docker-compose.yml` enable the commented lines for the Nginx service. Please be aware of the correct indentation so that `nginx` is recognized as an additional container. Also adjust the `PORTAL_BASE_URL` environment variable.
- Within the directory `resource/nginx/ssl/<Your server's DNS>` provide the necessary certificate files and a DH param file for your server.
- Within the files `docker-compose.yml` and `resource/nginx/conf.d/default.conf` replace all occurrences of `example.unitedplanet.de` with your desired DNS.
- If you changed the portal name (by modifying the EVN Var PORTAL_NAME) also adjust the htmlroot path in `resource/nginx/conf.d/default.conf`.

If the portal is contained in the separate portal-data volume (case 2 above), you are done now. If you mounted the portal from a local directory instead (case 1 above), adjust the volume definition in the Nginx service in `docker-compose.yml` as well:

```yml

nginx:
  ...
  volumes:
  ...
    - /portal/dir/on/local/system:/opt/intrexx/org
```

If your deployment is already running, you may now use the following commands to start your Nginx frontend:

```bash
docker compose up --no-start nginx
docker compose start nginx
```

If the deployment is not already running, use `docker compose up -d` instead as above and the Nginx service will be started alongside your deployment.

## Configure Solr credentials
Solr expects the Solr user credentials to be provided as base64 encoded hash within the `security.json`. Because Solr is not very transparent on how to derive the password string, this demo deployment is not able to create it. A tool like [solrpasswordhash](https://github.com/ansgarwiechers/solrpasswordhash) could be used to generate the string and adjust the `security.json` accordingly. Otherwise the default credentials need to be used.

## Custom configuration work
Starting with versions 10.0.10 and 10.4.0, it is possible for the user to provide additional initialization scripts, placed under `/entrypoint.d/`. Any `*.sh` script found there, will be executed by the `docker-entrypoint.sh` routine, right before the start of the portal. Please do not replace the entire directory as needed initialization scripts are already stored there in the original image provided by United Planet. Instead use a Dockerfile and the COPY command, to store your scripts in the directory.

# Distributed mode (horizontal scaling)
It is possible, to start the deployment in distributed mode when on version 10.15.0 or above. For this, a few adjustments in the `.env` file need to be made and a load balancer is needed.

All forementioned use cases apply for distributed use in the same way. In both cases (standalone and distributed) a init container is used to prepare the portal.

Note that switching a deployment between standalone and distributed is not simply possible in any direction. If you need to do this, please export the portal and redeploy.

## Adjustments to enable distributed mode
The `IX_DEPLOY_MODE` needs to be switched from `global` to `replicated`. The desired count of replications needs to be specified using `IX_DEPLOY_REPLICATIONS`.

`IX_DISTRIBUTED` needs to be true and `IX_DISTRIBUTED_NODELIST` needs to hold a list of all cluster nodes. For the docker compose example this may be a list of container names as in the example value.

## Port Binding
If you want to bind the Web, REST and ODATA Ports to your host system, you need to provide port ranges, instead of concrete ports in the distributed case as shown in the `.env.example` file.
Note that in some cases (Docker Desktop for Windows) Docker Compose incorrectly assumes ports as allocated. This issue can be resolved by either restarting the affected containers manually or changing the exposed ports to allocate any free port in the `docker-compose.yml` like so:
```yaml
    ports:
      - "1337"
      - "8101"
      - "9090"
```

## Load Balancing
Any kind of loadbalancer may be used to distributed traffic between the cluster members. The most simple way is using an nginx as shown in the `docker-compose.yml`. Note that only 1 nginx service should be used but 2 examples are given.

# Tags

The Intrexx images are tagged with the semantic version of the release. The semantic version contains 3 parts:

- Major version
- Minor version
- Patch level (online update number)

So, for example, the semantic version of Intrexx 21.03 OU05 would be 10.0.5.

Additionally for every major version, there is a specific latest tag, e.g. `21.03-latest`. This may be used to automatically receive the most recent patches but to avoid major version upgrades.

Please refer to the [Docker Hub page](https://hub.docker.com/r/unitedplanet/intrexx) for a complete list of available tags.

# List of environment vars
Name | Default value | Description
--- | --- | ---
`PORTAL_NAME` | portal | The name of the portal.
`PORTAL_BASE_URL` | http://localhost:1337/ | The base URL of the portal.
`DB_CREATE` | true | Create a new DB (true) or use existing (false). Available from version 10.0.10 and 10.4.0 onwards.
`DB_HOST` | db | Hostname of the database server.
`DB_PORT` | 5432 | Port of the database.
`DB_NAME` | ixportal | Name of the database.
`DB_USER` | postgres | Username for database authentication.
`DB_PASSWORD` | .admin1 | Password for database authentication.
`IX_DISTRIBUTED` | false | If Intrexx should run in distributed mode (horizontal scaling).
`IX_DISTRIBUTED_NODELIST` | "" | List of all nodes in a distributed cluster. Either DNS (or container names) or IP.
`SOLR_HOST` | solr | Hostname of the SOLR zookeeper server.
`SOLR_USER` | solr | Username for zookeeper connection. Usually the default credentials need to be used. See section "Configure Solr credentials" above.
`SOLR_PASSWORD` | SolrRocks | Password for zookeeper connection. Usually the default credentials need to be used. See section "Configure Solr credentials" above.
`SOLR_PORT` | 9983 | Port of the SOLR zookeeper.
`SOLR_SASL_DISABLED` | true | If SOLR is run without SASL, this parameter adjusts the intrexx search client accordingly.
`TEMPLATE_PATH` | /opt/intrexx/orgtempl/blank | The directory ultimately used as the portal template. **ATTENTION** In most cases this does not need to be adjusted, because it is adjusted by the entrypoint skript, if a portal template is provided by zip file.
`PORTAL_ZIP_NAME` | portal-template.zip | Optional name of the portal export zip, that should be used as a template.
`PORTAL_ZIP_MNTPT` | /opt/intrexx/org | Optional mountpoint of a volume containing the export zip in the container. 
`TOMCAT_SEC_HEADER_XUSER` | deny | Can be set to allow to change the according value in the web.xml. Available from version 10.0.10 and 10.4.0 onwards.
`TOMCAT_SEC_HEADER_XDOMAIN` | deny | Can be set to allow to change the according value in the web.xml. Available from version 10.0.10 and 10.4.0 onwards.
`TOMCAT_SEC_HEADER_XKRBTICKET` | deny | Can be set to allow to change the according value in the web.xml. Available from version 10.0.10 and 10.4.0 onwards.
`TOMCAT_SEC_HEADER_XACCOUNTNAME` | deny | Can be set to allow to change the according value in the web.xml. Available from version 10.0.10 and 10.4.0 onwards.
`TOMCAT_SEC_HEADER_XFORWARDEDFOR` | deny | Can be set to allow to change the according value in the web.xml. Available from version 10.0.10 and 10.4.0 onwards.
`TOMCAT_SEC_HEADER_FORWARDED` | deny | Can be set to allow to change the according value in the web.xml. Available from version 10.0.10 and 10.4.0 onwards.
`TOMCAT_SEC_HEADER_XREALIP` | deny | Can be set to allow to change the according value in the web.xml. Available from version 10.0.10 and 10.4.0 onwards.
`TOMCAT_SEC_HEADER_XFORWARDEDHOST` | deny | Can be set to allow to change the according value in the web.xml. Available from version 10.0.10 and 10.4.0 onwards.
`TOMCAT_SEC_HEADER_XFORWARDEDPROTO` | deny | Can be set to allow to change the according value in the web.xml. Available from version 10.0.10 and 10.4.0 onwards.
`TOMCAT_SEC_HEADER_XORIGINALURL` | deny | Can be set to allow to change the according value in the web.xml. Available from version 10.0.10 and 10.4.0 onwards.
`TOMCAT_SEC_HEADER_RECEIVEONNONLOOPBACKINTERFACE` | false | Can be set to true to change the according value in the web.xml. Available from version 10.0.10 and 10.4.0 onwards.

# FAQ

## Is this deployment scalable?

Yes, starting with version 10.15.0. See above on how.

## Can I have multiple portals?

No. This deployment runs exactly one Intrexx portal. For additional portals, please make additional deployments.

## How can I connect with the portal manager?

As there is only one portal per deployment, a supervisor is not necessary and therefore not included. As a consequence, a connection to the portal can not be established via a supervisor (i.e. port 7960) as usual. Instead, it must must be connected [directly to the portal](https://onlinehelp.unitedplanet.com/intrexx/10000/de-de/Content/Onlinehelp-Intrexx/helpfiles/de.uplanet.lucy.client.lucymanager.portaldirect.ConnectPage.html). On a local machine (e.g. for development purposes) the portal can be reached either through the host address 0.0.0.0 (alternatively the local IP) and the default port 8101.

## Where do I find the logs?

A combined log from all containers can be obtained from Docker compose directly via

```bash
docker compose logs -f
```

Alternatively, the separate logs are available directly from the containers via

```bash
docker compose logs -f [intrexx|db|solr|nginx]
# e.g.: docker compose logs -f intrexx
```

## Can I use another database type?

Currently, only PostgreSQL is supported out of the box. While it is possible to modify deployment and image to use another database, no support is provided by United Planet.

## Can I use this in production?

Yes. We offer full support when encountering problems caused from our side in the deployment of Intrexx in Docker. We continue to work steadily on improving our scripts in order to enable a deployment that runs as smooth as possible.

