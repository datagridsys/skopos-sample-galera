<!-- vim: set filetype=markdown: -->

## Skopos Sample MariaDB Galera Cluster for Docker Swarm

### Introduction

This is a sample application which can be used to deploy a MariaDB Galera Cluster to Docker Swarm:  this is a multi-master cluster for MariaDB which supports the XtraDB/InnoDB storage engines on Linux.  For more information please see [what is MariaDB Galera cluster](https://mariadb.com/kb/en/mariadb/what-is-mariadb-galera-cluster/).

According to the use case, there are many ways to deploy a clustered database to swarm.  For simplicity, this sample implementation uses these three services:

* `seed`:  the galera seed node is used to bootstrap the cluster.  It is the source of the original data with which the newly created galera nodes sync, and it is thereafter discarded (alternatively it may be re-purposed to join the new galera cluster as a node).  The seed is configured to create a user `my_user` with password `my_password` with access to database `my_database`.
* `cluster`:  the galera cluster nodes deploy as a single global service (one galera node per swarm node) with a further constraint that each usable swarm node also have the label `db==yes` (this can be used to control the total number of galera nodes, and which swarm nodes they deploy to).  _The sample app requires at least two swarm nodes to be assigned this label._
* `phpmyadmin`:  this singleton service exposes a database administration web interface on port `10000` of the swarm.  _phpMyAdmin_ is configured to allow connecting to an arbitray database server, so it can be used to administer any of the galera nodes.  The galera cluster API is also exposed on port `10001` of the swarm and can be accessed with the mysql command line client.

The galera cluster seed and its nodes all use the public docker image `colinmollenhour/mariadb-galera-swarm:latest`.  Please see this [docker hub](https://hub.docker.com/r/colinmollenhour/mariadb-galera-swarm/) page for details.

Below is a view of the sample application model:

![model-bootstrap](img/model-bootstrap.png)

### Application Files

The application includes a model descriptor `model.yaml`, and two environment descriptors which may be used in combination:

* `env.yaml`:  the base environment file common to all deployments.  It specifies variables, plugin configuration, and quality gates for the galera nodes.  The quality gates use Skopos probes to verify the successful deployment each galera node:
    * [http probe](https://github.com/opsani/probe-http):  verify the galera image built-in healthcheck on port `8080`.
    * [mariadb probe](https://github.com/opsani/probe-mariadb) verify the user created with the seed can access the mysql API and the created database `my_database` on the application network.
* `env.bootstrap.yaml`:  an environment file which simply sets the galera seed node replicas count to `1` (instead of the default `0`).  This file is used when bootstrapping a galera cluster.

### Usage Notes

* The seed in this sample app uses a randomly named volume on instance creation for its mariadb seed data. Each galera cluster node uses a volume named `galera-node-db` for its data.
* Swarm node preparation:  label two or more nodes of the swarm with `db=yes` (e.g., `docker node update --label-add db=yes <node-id>`).  A galera cluster node will be created on each labeled swarm node.  The cluster seed will be created on a swarm manager node.
* To bootstrap the galera cluster, deploy using both environment files.  To remove the seed node, or upgrade the cluster, omit `env.bootstrap.yaml`.

### Quick Start Example

This is a stream-lined quick-start example.  See the [Skopos documentation](http://doc.opsani.com/skopos/edge/) for detailed information.

Install the `skopos` command line utility on a swarm manager node as `~/bin/skopos`:
```
wget https://s3.amazonaws.com/get-skopos/edge/linux/skopos
chmod +x skopos
mkdir -p ~/bin
mv skopos ~/bin
```

Start the Skopos engine on the same node (example binds to the default port 8100):

```
docker run -d -p 8100:8100 --restart=unless-stopped --name skopos \
-v /var/run/docker.sock:/var/run/docker.sock opsani/skopos:edge
```

Label **two or more** swarm nodes as described above.  Verify the swarm provides access to ports 10000-10001 to the outside world (so you can access the sample application web UIs with a browser after it is deployed).

Load the sample application into Skopos directly from github:

```
~/bin/skopos load --project galera \
--env github://opsani/skopos-sample-galera/env.yaml \
--env github://opsani/skopos-sample-galera/env.bootstrap.yaml \
github://opsani/skopos-sample-galera/model.yaml
```

The Skopos UI, accessible on port 8100 of the manager node, now shows the `galera` application on the application list.  Click to open this application - showing its model ready for the bootstrap deploy.

* Click `Start` to start the deployment
* Click `Switch to Plan` to see a logical flow diagram of the deployment plan.  This plan, like the model, visually indicates progress and status during deployment.  Click on the `seed` service to zoom-in to the component plan details.  Here you can see the quality probes which validate the successful deployment of the seed.

Once the cluster is deployed, the seed may be destroyed by deploying without the bootstrap environment descriptor:

```
~/bin/skopos load --project galera \
--env github://opsani/skopos-sample-galera/env.yaml \
github://opsani/skopos-sample-galera/model.yaml
```

### Verification

A quick verification of the deployed galera cluster involves creating a table in database `my_database` on one galera node, and verifying this table is also created (sync'ed) on the other galera nodes (including the seed, if it is running).

**Access with phpMyAdmin**

The web interface for phyMyAdmin is accessible on port `10000` of the swarm.  Log in requires the user name `my_user`, password `my_password`.  Because phpMyAdmin is configured via an environment variable in the model to allow access to an arbitrary database server, you must also supply an IP address or DNS name for a server as indicated below:

* To access mariadb on the seed use server:
    * `seed`:  this resolves to the seed service, which has a single instance, and so provides access to the seed node of the galera cluster.  You can also use `tasks.seed`.
* To access mariadb on an arbitrary galera cluster node use server:
    * `cluster`:  this resolves to the cluster service, and will provide access to whatever galera node is resolved by the swarm routing mesh.  You can also use `tasks.cluster`.
* To access mariadb on a particular galera cluster node use server:
    * `<service.slot.taskid>`:  this resolves to a particular galera node as indicated.  The values for `service`, `slot` and `taskid` can be obtained from the output of `docker service ps --no-trunc galera_cluster` where `service.slot` is represented as the _NAME_ of the task (e.g. `galera_cluster.t9ewvghu1sjsju3utcc9n6d5l`) and `taskid` is represed as the un-truncated _ID_ (e.g. `r1absmv3wg6hr77o86l18vzbp`).  You can also use the _CONTAINER ID_ for the galera node returned by `docker ps` on a particular swarm node.

**Access with the mysql cmdline client**

The mariadb API for the galera cluster is exposed on port `10001` of the swarm (resolves to an arbitrary galera cluster node).  This API can be accessed with a mysql client (e.g., `apt-get install mysql-client` on Ubuntu/Debian or `yum install mysql` on CentOs/RedHat).  

To connect to the mariadb API on an arbitray galera node run `mysql -u my_user --host <swarm_IP> --port 10001 --password=my_password` from the command line of any host with network access to the swarm.  Within the mysql client cmdline interface, execute the following commands to create a new table `my_table` in the database `my_database`, and to show the hostname of the galera cluster node servicing the request (this is the same as the container instance ID):

```
use my_database;
create table my_table (my_column int);
show tables;
show variables where variable_name = 'hostname';
```
