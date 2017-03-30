<!-- vim: set filetype=markdown: -->

### Skopos Sample MariaDB Galera Cluster for Docker Swarm

The model and environment files for this application can be used to deploy a Mariadb Galera cluster to Docker Swarm.

This is a test application which publishes each galera node's mariadb port (3306) on the host it is running (`PublishMode: host`), rather than publishing this port for the service using the swarm routing mesh.  Docker 17.03.0-ce suffers from docker issue [31249](https://github.com/docker/docker/issues/31249) - two services constrained to different swarm nodes cannot expose the same port using mode=host. Because of this, each of the four galera node services of this app uses a different port in the range 10000-10003.

To change this behavior and use the swarm routing mesh instead of a single host port map:

* Remove or comment out from the model the separate per-node gateways section
* Uncomment the gateways section at the end of the model to expose the galera nodes on the swarm routing mesh.

The application includes several environment files used in combination to bootstrap or upgrade the application.

* `env.yaml`:  the base environment file common to all deploys.  It defines plugins, volume names and variables.
* `env.ports.yaml`:  an environment which defines the port to be exposed for each galera node on its host.  Different copies of this file may be used to deploy multiple galera clusters to the same swarm.
* `env.bootstrap.yaml`:  an environment which simply sets the galera seed node replicas count to 1.  This file is used when bootstrapping a galera cluster.

### Usage Notes

* The seed uses a randomly named volume on instance creation for its mariadb seed data.
* The seed, and each node, is created as its own service, and each is represented by its own skopos component.  Each of these is constrained to be placed on a swarm node matching a label.  
* Swarm node preparation:
    * Label one or more nodes of the swarm with `gseed` (e.g., `docker node update --label-add gseed=1 <node-id>`).  The seed will be created on one of the labeled nodes.
    * Divide the swarm nodes into non-overlapping pairs.  Label four of these pairs so that each pair has one of these labels: `gpair0`, `gpair1`, `gpair2`, `gpair3`.  Each galera node is constrained to its labeled pair of swarm nodes, so that if a swarm node fails, the galera node is re-started on the other node of its pair where it may sync its data and recover the galera cluster.
* To bootstrap the galera cluster, deploy using all three environment files.  To remove the seed node, or upgrade the cluster, omit `env.bootstrap.yaml`.

### Quick Start Example

Pull the Skopos Docker image to a swarm manager node:  
```
docker pull datagridsys/skopos:beta
```

Extract and install the control utility as `~/bin/sks-ctl`:
```
mkdir -p ~/bin
rm -rf ~/bin/sks-ctl
touch ~/bin/sks-ctl
docker run                               \
    --rm                                 \
    -v ~/bin/sks-ctl:/skopos/bin/sks-ctl \
    --entrypoint "/bin/bash"             \
    datagridsys/skopos:beta              \
    -c 'cp -a engine /skopos/bin/sks-ctl'
```

Start the skopos engine (example binds to port 8090):

```
docker run -d --net=host --restart=unless-stopped --name skopos   \
    -v /var/run/docker.sock:/var/run/docker.sock                  \
    datagridsys/skopos:beta --bind localhost:8090
```

Clone this repo to a working directorory on the same manager node:
```
git clone https://github.com/datagridsys/skopos-sample-galera.git
```

Label the swarm nodes as described above.  Verify all swarm node hosts expose the ports 10000-10003 to the outside world.

From the directory containing the application model and environment files, load the application:
```
~/bin/sks-ctl -bind :8090 load -project galera -env env.yaml -env env.ports.yaml -env env.bootstrap.yaml model.yaml
```

The Skopos UI, accessible on port 8090 of the manager node, now displays the model ready for its bootstrap deploy.  Once the cluster is deployed, the seed may be destroyed with:
```
~/bin/sks-ctl -bind :8090 load -project galera -env env.yaml -env env.ports.yaml model.yaml
```

### Verification

A quick verification:

* Docker exec into and run the mysql client on one of the galera nodes:  `docker exec -ti <id> mysql -u root -p`.  Provide the password `asdf` from the env file.
* Create a user with remote access permissions (*not secure*):
    * `CREATE USER 'qwer'@'%' IDENTIFIED BY 'qwer';`
    * `GRANT ALL ON *.* TO 'qwer'@'%';`
* From a remote host with the mysql client installed, access one of the galera nodes:  `mysql -u qwer --host <ip> --port <port> -p`  Provide the password `qwer`.
* Verify the galera cluster syncs:  create a database on one node and verify this change propagates to all nodes.