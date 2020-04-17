# Standalone Installation

To run a new instance of the TestBrain service in a docker container:

1. [Pull TestBrain Image](#pull-testBrain-image)
2. [Create and Configure Directories](#create-and-configure-directories)
3. [Start TestBrain Docker Container](#run-testbrain-docker-container)
4. [Update new version](#update-new-version)
5. [Create organization](#create-organization)
6. [Full example](#full-example)

## Pull TestBrain Image
Pull an image of the latest TestBrain version from the 
[Docker TestBrain Repository](https://cloud.docker.com/repository/docker/whenessel/testbrain):

```bash
docker pull whenessel/testbrain:<version>

```

\<version\> is a full version number of a TestBrain build. 
[See the complete list of available versions](https://cloud.docker.com/repository/docker/whenessel/testbrain/tags).


## Create and Configure Directories

TestBrain image container is a `stateful container`. Thus, before running it, you should create specific 
directories `on the host machine` to store TestBrain database, configuration files, logs and pass them to the 
TestBrain container as volumes. Otherwise, you risk losing data when the container is removed.

These directories are:

- \<storage\>/var/lib/postgresql/data — a directory where TestBrain stores its database. For a new installation, 
this directory must be empty.
- \<storage\>/var/lib/rabbitmq — a directory where TestBrain queue. For a new installation, 
this directory must be empty.
- \<storage\>/logs — a directory where TestBrain stores its log files.
- \<storage\>/storage — a directory where TestBrain stores ML models and VCS repositories.

\<storage\> is a base storage directory. We recommend `/opt/testbrain`.

Set permission for shared storage
```bash
chmod -R 777 <storage>/{logs,storage}

```

## Run PostgreSQL Docker Container
Use the following command to create and run container with PostgreSQL service and map PostgreSQL data volumes and port.

```bash
docker create --name <postgresql-server-instance> \
-p <port on host>:5432 \
--restart=always \
--memory 8g \
--cpus 4 \
-v <storage>/var/lib/postgresql/data:/var/lib/postgresql/data \
-t postgres:10.6 \
postgres

docker start <postgresql-server-instance>
docker exec -it <postgresql-server-instance> createdb -h localhost -U postgres testbrain

```

Where:
- --name \<postgresql-server-instance\> - the arbitrary name for the container.
- -v \<storage\>/var/lib/postgresql/data:/var/lib/postgresql/data - binding the PosqgreSQL data directory on the host 
machine to the respective /var/lib/postgresql/data directory inside the container.
- -p \<port on host\>:5432 -  parameter that defines the ports mapping. It instructs the host machine to listen on port 
<port on host> and propagate all traffic to the port `5432` inside the docker container. 
You can set `0.0.0.0:5432:5432` for bind to all external interfaces on host machine.

>It is required to adjust the parameters of the postgres service in the container. 
>Run `docker exec -it <postgresql-server-instance> /bin/bash` and open 
>configuration file `nano /var/lib/pgsql/10/data/postgresql.conf`.
>
>Example params for 2-4 cpu cores, 8GB ram
>```
>max_connections = 256
>shared_buffers = 1024MB
>work_mem = 4MB
>maintenance_work_mem = 256MB
>dynamic_shared_memory_type = posix
>effective_io_concurrency = 2
>max_worker_processes = 2
>max_parallel_workers_per_gather = 1
>max_parallel_workers = 2
>wal_buffers = 16MB
>max_wal_size = 4GB
>min_wal_size = 1GB
>checkpoint_completion_target = 0.9
>random_page_cost = 4.0
>effective_cache_size = 4GB
>default_statistics_target = 100
>```

## Run RabbitMQ Docker Container

```bash
docker create --name <rabbitmq-server-instance> \
-p <port on host>:5672 \
--restart=always \
--memory 1024m \
--cpus 1 \
-v <storage>/var/lib/rabbitmq:/var/lib/rabbitmq \
-t rabbitmq:3.7.14-management

docker start <rabbitmq-server-instance>

```

Where:
- --name \<rabbitmq-server-instance\> - the arbitrary name for the container.
- -v \<storage\>/var/lib/rabbitmq:/var/lib/rabbitmq - binding the RabbitMQ data directory on the host machine to the 
respective /var/lib/rabbitmq directory inside the container.
- -p \<port on host\>:5672 -  parameter that defines the ports mapping. It instructs the host machine to listen on 
port <port on host> and propagate all traffic to the port `5672` inside the docker container. 
You can set `0.0.0.0:5672:5672` for bind to all external interfaces on host machine.


## Run TestBrain Docker Container
### Run WEB TestBrain Instance

```bash
docker create --name <testbrain-web-server-instance> \
-p <port on host>:80 \
--restart=always \
--link <postgresql-server-instance>:<postgresql-server-hostname> \
--link <rabbitmq-server-instance>:<rabbitmq-server-hostname> \
-e ENV_NAME='web' \
-e RDS_DB_NAME='testbrain' \
-e RDS_USERNAME='<postgres-server-username>' \
-e RDS_PASSWORD='<postgres-server-password>' \
-e RDS_HOSTNAME='<postgres-server-hostname>' \
-e BROKER_URL='amqp://guest:guest@<rabbitmq-server-hostname>:5672//' \
-e DJANGO_SETTINGS_MODULE='testbrain.settings.standalone' \
-e BASE_SITE_DOMAIN='<fqdn hostname>' \
-e BASE_ORG_DOMAIN='<fqdn hostname>' \
-v <storage>/storage:/app/efs \
-v <storage>/logs:/app/logs \
-t whenessel/testbrain:<version>

docker start <testbrain-web-server-instance>

```
**To be sure of the correct sending of email alerts, use an external mail server.**
Example GMAIL
```bash
-e EMAIL_HOST='smtp.gmail.com' \
-e EMAIL_HOST_USER='testbrain-no-reply@gmail.com' \
-e EMAIL_HOST_PASSWORD='<PASSWORD>' \
-e EMAIL_PORT=587 \
-e EMAIL_USE_TLS=True \
```
You can use enterprise or personal email server.

Where:
- --name \<testbrain-web-server-instance\> - the arbitrary name for the container.
- --link \<postgresql-server-instance\>:\<postgresql-server-hostname\> Creates a connection between containers 
through the internal network between containers within the same docker service. WEB TestBrain Instance can access 
<postgresql-server-instance> by hostname <postgresql-server-hostname>. 
`example. --link testbrain_postgresql:postgres_hostname` and use `postgres_hostname` as `<postgres-server-hostname>`.
- --link \<rabbitmq-server-instance\>:\<rabbitmq-server-hostname\> Creates a connection between containers through 
the internal network between containers within the same docker service. WEB TestBrain Instance can access 
<rabbitmq-server-instance> by hostname <rabbitmq-server-hostname>. 
`example. --link testbrain_rabbitmq:rabbitmq_hostname` and use `rabbitmq_hostname` as `<rabbitmq-server-hostname>`.
- -v \<storage\>/storage:/app/efs - binding the WEB TestBrain instance data directory on the host machine to the 
respective /app/efs directory inside the container.
- -p \<port on host\>:80 -  parameter that defines the ports mapping. It instructs the host machine to listen on port 
<port on host> and propagate all traffic to the port `80` inside the docker container. 
You can set 0.0.0.0:80:80 for bind to all external interfaces on host machine.
- -e `BASE_SITE_DOMAIN='<fqdn hostname>'` and -e `BASE_ORG_DOMAIN='<fqdn hostname>'` -  Replace `<fqdn hostname>` with 
the host name or domain of the internal network (intranet) or global domain name (internet). As an example: 
Your organization’s domain on the Internet is `example.com`, you must specify `example.com`, after installation you will 
need to create an organization (artifact from the cloud version) and you will need to specify subdomain 
(if you specify `testbrain`), you can access the system at `http://testbrain.example.com/`. For intranet, everything 
is the same: BASE_SITE_DOMAIN = BASE_ORG_DOMAIN = `example.corp.intra` and when creating the organization specify 
`testbrain-example`, the system will have the address `http://testbrain-example.example.corp.intra/`.
> If the postgresql server and rabbitmq are physically located elsewhere, then `--link` is not required.

#### First time run
```bash
docker exec <testbrain-web-server-instance> python manage.py collectstatic --noinput
docker exec <testbrain-web-server-instance> python manage.py migrate --noinput

```


### Run WORKER TestBrain Instance

```bash
docker create --name <testbrain-worker-server-instance> \
--restart=always \
--link <postgresql-server-instance>:<postgresql-server-hostname> \
--link <rabbitmq-server-instance>:<rabbitmq-server-hostname> \
-e ENV_NAME='worker' \
-e RDS_DB_NAME='testbrain' \
-e RDS_USERNAME='<postgres-server-username>' \
-e RDS_PASSWORD='<postgres-server-password>' \
-e RDS_HOSTNAME='<postgres-server-hostname>' \
-e BROKER_URL='amqp://guest:guest@<rabbitmq-server-hostname>:5672//' \
-e DJANGO_SETTINGS_MODULE='testbrain.settings.standalone' \
-e BASE_SITE_DOMAIN='<fqdn hostname>' \
-e BASE_ORG_DOMAIN='<fqdn hostname>' \
-v <storage>/storage:/app/efs \
-v <storage>/logs:/app/logs \
-t whenessel/testbrain:<version>

docker start <testbrain-worker-server-instance>

```

**To be sure of the correct sending of email alerts, use an external mail server.**
Example GMAIL
```bash
-e EMAIL_HOST='smtp.gmail.com' \
-e EMAIL_HOST_USER='testbrain-no-reply@gmail.com' \
-e EMAIL_HOST_PASSWORD='<PASSWORD>' \
-e EMAIL_PORT=587 \
-e EMAIL_USE_TLS=True \
```
You can use enterprise or personal email server.

Where:
- --name \<testbrain-web-server-instance\> - the arbitrary name for the container.
- --link \<postgresql-server-instance\>:\<postgresql-server-hostname\> Creates a connection between containers 
through the internal network between containers within the same docker service. WEB TestBrain Instance can access 
<postgresql-server-instance> by hostname <postgresql-server-hostname>. 
`example. --link testbrain_postgresql:postgres_hostname` and use `postgres_hostname` as `<postgres-server-hostname>`.
- --link \<rabbitmq-server-instance\>:\<rabbitmq-server-hostname\> Creates a connection between containers through 
the internal network between containers within the same docker service. WEB TestBrain Instance can access 
<rabbitmq-server-instance> by hostname <rabbitmq-server-hostname>. 
`example. --link testbrain_rabbitmq:rabbitmq_hostname` and use `rabbitmq_hostname` as `<rabbitmq-server-hostname>`.
- -v \<storage\>/storage:/app/efs - binding the WEB TestBrain instance data directory on the host machine to the 
respective /app/efs directory inside the container.

> If the postgresql server and rabbitmq are physically located elsewhere, then `--link` is not required.


## Update new version
#### 1. Pull needed version TestBrain docker image from repository
```bash
docker login
docker pull whenessel/testbrain:<version>

```

#### 2. Stop currenly running TestBrain docker containers.
```bash
docker stop <testbrain-web-server-instance>
docker stop <testbrain-worker-server-instance>

``` 

#### 3. Remove old TestBrain docker containers.
```bash
docker rm <testbrain-web-server-instance>
docker rm <testbrain-worker-server-instance>

```

#### 4. Run TestBrain docker container
*WEB Testbrain Instance*

```bash
docker create --name <testbrain-web-server-instance> \
-p <port on host>:80 \
--restart=always \
--link <postgresql-server-instance>:<postgresql-server-hostname> \
--link <rabbitmq-server-instance>:<rabbitmq-server-hostname> \
-e ENV_NAME='web' \
-e RDS_DB_NAME='testbrain' \
-e RDS_USERNAME='<postgres-server-username>' \
-e RDS_PASSWORD='<postgres-server-password>' \
-e RDS_HOSTNAME='<postgres-server-hostname>' \
-e BROKER_URL='amqp://guest:guest@<rabbitmq-server-hostname>:5672//' \
-e DJANGO_SETTINGS_MODULE='testbrain.settings.standalone' \
-e BASE_SITE_DOMAIN='<fqdn hostname>' \
-e BASE_ORG_DOMAIN='<fqdn hostname>' \
-v <storage>/storage:/app/efs \
-v <storage>/logs:/app/logs \
-t whenessel/testbrain:<version>

docker start <testbrain-web-server-instance>
docker exec <testbrain-web-server-instance> python manage.py collectstatic --noinput
docker exec <testbrain-web-server-instance> python manage.py migrate --noinput

```

*WORKER Testbrain Instance*

```bash
docker create --name <testbrain-worker-server-instance> \
--restart=always \
--link <postgresql-server-instance>:<postgresql-server-hostname> \
--link <rabbitmq-server-instance>:<rabbitmq-server-hostname> \
-e ENV_NAME='worker' \
-e RDS_DB_NAME='testbrain' \
-e RDS_USERNAME='<postgres-server-username>' \
-e RDS_PASSWORD='<postgres-server-password>' \
-e RDS_HOSTNAME='<postgres-server-hostname>' \
-e BROKER_URL='amqp://guest:guest@<rabbitmq-server-hostname>:5672//' \
-e DJANGO_SETTINGS_MODULE='testbrain.settings.standalone' \
-e BASE_SITE_DOMAIN='<fqdn hostname>' \
-e BASE_ORG_DOMAIN='<fqdn hostname>' \
-v <storage>/storage:/app/efs \
-v <storage>/logs:/app/logs \
-t whenessel/testbrain:<version>

docker start <testbrain-worker-server-instance>

```


## Create organization
For the proper functioning of the system, you need to create an organization.
```bash
docker exec -it <testbrain-web-server-instance> python manage.py createorganization

```

The request credentials. Most of these parameters are required for url generation to work properly.

```bash
Organization name:
Organization subdomain (leave blank to use '<slug from organization name>'): 
Username (leave blank to use 'root'):         
Email address: 
Password: 
Password (again): 

```

- `Organization name` - Organization name
- `Organization subdomain` - Organization subdomain, if blank system use slug function for create it. subdomain used for
url generation and tenant a few organization with one system instance.
- `Username` - organization admin username. DON'T use `admin`, `root`, etc.


Finish.


## Full Example
This example used on MacOS, but should work on other Linux distributions.

```bash

mkdir testbrain
cd testbrain/


mkdir -p var/lib/postgresql/data
mkdir -p var/lib/rabbitmq
mkdir {logs,storage}
chmod -R 777 {logs,storage}


docker create --name testbrain_test_postgres \
-p 5432:5432 \
--memory 8g \
--cpus 4 \
-v $(pwd)/var/lib/postgresql/data:/var/lib/postgresql/data \
-t postgres:10.6 postgres

docker start testbrain_test_postgres
docker exec -it testbrain_test_postgres createdb -h localhost -U postgres testbrain


docker create --name testbrain_test_rabbitmq \
-p 5672:5672 \
--memory 1024m \
--cpus 1 \
-v $(pwd)/var/lib/rabbitmq:/var/lib/rabbitmq \
-t rabbitmq:3.7.14-management
docker start testbrain_test_rabbitmq

docker create --name testbrain_test_web \
-p 0.0.0.0:80:80 \
--link testbrain_test_postgres:testbrain_test_postgres \
--link testbrain_test_rabbitmq:testbrain_test_rabbitmq \
--memory 4g \
--cpus 2 \
-e ENV_NAME='web' \
-e RDS_DB_NAME='testbrain' \
-e RDS_USERNAME='postgres' \
-e RDS_PASSWORD='' \
-e RDS_HOSTNAME='testbrain_test_postgres' \
-e BROKER_URL='amqp://guest:guest@testbrain_test_rabbitmq:5672//' \
-e DJANGO_SETTINGS_MODULE='testbrain.settings.standalone' \
-e BASE_SITE_DOMAIN='iMac-Pro-User.local' \
-e BASE_ORG_DOMAIN='iMac-Pro-User.local' \
-v $(pwd)/storage:/app/efs \
-v $(pwd)/logs:/app/logs \
-t whenessel/testbrain:latest

docker start testbrain_test_web


docker exec testbrain_test_web python manage.py collectstatic --noinput
docker exec testbrain_test_web python manage.py migrate --noinput



docker create --name testbrain_test_worker \
--link testbrain_test_postgres:testbrain_test_postgres \
--link testbrain_test_rabbitmq:testbrain_test_rabbitmq \
--memory 4g \
--cpus 2 \
-e ENV_NAME='worker' \
-e RDS_DB_NAME='testbrain' \
-e RDS_USERNAME='postgres' \
-e RDS_PASSWORD='' \
-e RDS_HOSTNAME='testbrain_test_postgres' \
-e BROKER_URL='amqp://guest:guest@testbrain_test_rabbitmq:5672//' \
-e DJANGO_SETTINGS_MODULE='testbrain.settings.standalone' \
-e BASE_SITE_DOMAIN='iMac-Pro-User.local' \
-e BASE_ORG_DOMAIN='iMac-Pro-User.local' \
-v $(pwd)/storage:/app/efs \
-v $(pwd)/logs:/app/logs \
-t whenessel/testbrain:latest

docker start testbrain_test_worker



docker exec -it testbrain_test_web python manage.py createorganization
###
Organization name: <name>
Organization sub domain: <instance subdomain only> - <subdomain>.BASE_SITE_DOMAIN
Username (leave blank to use 'root'): <do not use root or admin>         
Email address: <email address>
Password: 
Password (again): 
Organization created successfully.

```


Required | Category | Variable Name | Default Value | Description
--- | --- | --- | --- | ---
No | SYSTEM | PYTHONUNBUFFERED | 1 | System variable.
No | SYSTEM | GIT_PYTHON_REFRESH | 'quiet' | System variable
Yes | ENVIRONMENT | ENV_NAME | 'web' | web - run web interface. worker - run celery workers.
Yes | ENVIRONMENT | DJANGO_SETTINGS_MODULE | 'testbrain.settings.standalone' | Define specify configuration file with Django Faramework variables.
Yes | DATABASE | RDS_DB_NAME | 'testbrain' | Define database name. 
Yes | DATABASE | RDS_USERNAME | 'postgres' | Define database username.
Yes | DATABASE | RDS_PASSWORD | '' | Define database password.
Yes | DATABASE | RDS_HOSTNAME | 'localhost' | Define database host/ip.
Yes | DATABASE | RDS_PORT | 5432 | Define database port.
Yes | RABBITMQ/CELERY | BROKER_URL | 'amqp://guest:guest@localhost:5672//' | Define RabbitMQ connection uri. 'amqp://<username>:<password>@<host>:<port>/<vhost>'
No | EMAIL NOTIFICATION | EMAIL_BACKEND | 'django.core.mail.backends.smtp.EmailBackend' | Define email send backend, see Django docs.
No | EMAIL NOTIFICATION | EMAIL_HOST | 'smtp.gmail.com' | Define mail server host. 
No | EMAIL NOTIFICATION | EMAIL_PORT | 587 | Define mail server port.
No | EMAIL NOTIFICATION | EMAIL_USE_TLS | 'True' | Define mail server TLS usage.
Yes | EMAIL NOTIFICATION | EMAIL_HOST_USER | '' | Define mail username.
Yes | EMAIL NOTIFICATION | EMAIL_HOST_PASSWORD | '' | Define mail user password.
No | EMAIL NOTIFICATION | DEFAULT_FROM_EMAIL | 'Noreply <noreply@appsurify.com>' | Define FROM string in messages.
No, only for amazon eb | AMAZON | REGION | '' | Define region specialy for Amazon deployment.
No, only for amazon eb | AMAZON | FILE_SYSTEM_ID | '' | Define EFS ID specialy for Amazon deployment.
Yes | ENVIRONMENT | MOUNT_DIRECTORY | '/app/efs' | Define path for storing cloned repositories.
Yes | ENVIRONMENT | BASE_SITE_NAME | 'Appsurify' | Define basically brand name.
Yes | ENVIRONMENT | BASE_SITE_DOMAIN | '' | Define base system domain.
Yes | ENVIRONMENT | BASE_ORG_DOMAIN | '' | Define base domain for organizations.
Yes | STRIPE | STRIPE_LIVE_MODE | False | Define True for use LIVE mode stripe integraiton.
Yes | STRIPE | STRIPE_LIVE_SECRET_KEY | 'sk_live_************************' | 
Yes | STRIPE | STRIPE_LIVE_PUBLIC_KEY | 'pk_live_************************' |
Yes | STRIPE | STRIPE_TEST_SECRET_KEY | 'sk_test_************************' |
Yes | STRIPE | STRIPE_TEST_PUBLIC_KEY | 'pk_test_************************' |
Yes | ENVIRONMENT | LOCAL_SECRET_KEY | '************************************************' | Define secret (random) string for webhooks integration.
Yes | ENVIRONMENT | GITHUB_SECRET_KEY | '***********************************************' | Define secret (random) string for webhooks integration.
