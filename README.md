<!-- vim: set filetype=markdown: -->

The model and env files for this application can be used to deploy a mariadb galera cluster to docker swarm:

* `model.bootstrap.yaml`: model to bootstrap the galera cluster.  This model will create the seed service first and then four galera nodes which sync to the seed.
* `model.yaml`: model to remove the seed or update the deployed cluster.  This model is identical to the bootstrap model except the seed replica count is zero.  The galera version is set to 1.0 using var substitution defaults.
* `env.yaml`:  env file which uses docker-swarm as the core plugin, defines volumes for the nodes, and defines vars for instance environment variables.
* `env-v2.yaml`:  env file which may be additionally loaded to set the galera image version to 2.0.

Usage notes:

* The seed uses a randomly named volume on instance creation for its mariadb seed data.
* The seed, and each node, is created as its own service, and each is represented by its own skopos component.  Each of these is constrained to be placed on a swarm node matching a label.  
* Setup:
    * Label one or more nodes of the swarm with `gseed=1` (e.g., `docker node update --label-add gseed=1 <node-id>`).  The seed will be created on one of the labeled nodes.
    * Divide the swarm nodes into non-overlapping pairs.  Label four of these pairs so that each pair has one of these labels: `gpair0=1`, `gpair1=1`, `gpair2=1`, `gpair3=1`.  Each galera node is constrained to its labeled pair of swarm nodes, so that if a swarm node fails, the galera node is re-started on the other node of its pair where it may sync its data and recover the galera cluster.
* To bootstrap the galera cluster, deploy using `model.bootstrap.yaml`.  After bootstrapping, deploy using `model.yaml` to remove the seed.
