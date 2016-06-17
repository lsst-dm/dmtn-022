..
  Uncomment this section and modify the DOI strings to include a Zenodo DOI badge in the README
  .. image:: https://zenodo.org/badge/doi/10.5281/zenodo.#####.svg
     :target: http://dx.doi.org/10.5281/zenodo.#####

Current status
==============

Current features are:

Docker images build
-------------------

Binary images
^^^^^^^^^^^^^

Qserv repository contains scripts which allows developers to build Docker images, on
their workstation, for:

* **Qserv release:** this container contains version of product `qserv_distrib` tagged
  `qserv_latest` on eups distribution server. This container is named
  `qserv/qserv:latest`.
* **Qserv current sprint version:** this container contains version of product `qserv_distrib` tagged
  `qserv-dev` on eups distribution server. Goal is to deliver a container with
  up-to-date Qserv dependencies to all Qserv developers. This container is named
  `qserv/qserv:dev`.
* **Qserv binaries built from a given git branch:** this container, based on `qserv/qserv:dev`,
  allows developers to easily package, deploy and test their code on heterogeneous infrastructures.

Configured images
^^^^^^^^^^^^^^^^^

Qserv container configuration script allows to build, on top of each binary images above, Qserv master and worker images.
These two Docker images are built with same binaries but different configurations.

For example, for `qserv/qserv:dev` image, two images named `qserv/qserv:dev_master` and
`qserv/qserv:dev_worker` will be built and pushed to Docker Hub.


Container deployment
--------------------

Qserv containers deployment is currently performed using shell scripts. Several
deployment procedures are available:

* **Development:** allow a developer to mount a host directory containing some LSST git repositories
  inside `qserv/qserv:dev` container. This procedure provides an identical development environment
  to whole team. It is not widely used currently.
* **Multinode:** allow to run master and worker containers on one host. This
  allows a developer to automatically launch multinode integration tests on its workstation.
* **Multinode for CI:** previous procedure is embedded inside travis-ci so
  that, for each Qserv commit, Docker images are build, launched and validated using multinodes integration tests.
* **Distributed multinode:** shmux, a parallel ssh tool, is used to deploy Qserv containers on
  each cluster machine. Only one container is deployed on each machine. Each machine also has a local MySQL data directory,
  which is mounted by its local Qserv container. This procedure has been extensively tested and used to deploy cutting-edge Qserv
  on IN2P3 cluster, for Large Scale Tests.

Tracks
======

Optimize container size
-----------------------

Qserv container size is currently around 2GB, composed of: 

* 135 MB for up-to-date system base image (Debian),
* 480 MB for Qserv system dependencies,
* 1450 MB for eups dependencies (comparison with bare-metal installation shows that no significant overhead is induced by containerization).

Some optimizations might be performed in order to reduce eups dependencies size or replace them with system dependencies.
For example eups package size for mariadb is currently ~600MB whereas system dependency install for mariadb is only ~135MB.

Continuous integration
----------------------

* **Publish on eups distribution server each new successful Qserv build** (use multinode integration tests to qualify it), and tag it *qserv-dev*. Then, create automatically:

  * a Docker image containing this build,
  * a master and a worker Docker image based on image above.
See https://jira.lsstcorp.org/browse/DM-5864.

* Remove automatically deprecated Qserv images from Docker Hub. For example, keep only
  containers for three latest releases and those containing ticket branches in
  progress.

Monthly Release
---------------

Qserv monthly release should deliver:

* Docker image for Qserv binaries
* Docker images for Qserv master and worker

This could be performed through `Continuous integration`_ procedure.

Configuration
-------------

Qserv configuration is composed of a directory structure which contains all information required for Qserv execution (among others configuration files,
MySQL meta-data structure, temporary, log and lock files, startup scripts and secrets). This directory structure is different for master and worker setups.

Qserv package embeds a configuration script which automatically generates and fills this directory structure, using a meta-configuration file.
Script which produces `Configured images`_ has to be improved in order to allow developers to easily change main configuration parameters, like log level or secrets.

Current Qserv secrets are:

* MySQL root password
* Data loader password

**Directory structure for configuration could be deployed as a docker volume, in order to ease its update. Nevertheless, this requires a sophisticated design.** Indeed:

* configuration has to be generated using Qserv configuration script. This tool is currently embedded inside Qserv installation,
* compatibility between Qserv containers and configuration volume has to be checked at runtime.

Data storage
------------

At IN2P3, MySQL data is stored on host machine, and not inside the container.
This setup enable Qserv containers to serve PetaScale datasets.
So, **MySQL data for LSST should be**:

* **stored on host machine**, which would provide access to large storage infrastructure (local, or even remote)
* then mounted as a Docker volume by Qserv master or worker container.

**Huge improvments must be performed to ease this setup.** 

MySQL meta-data 
^^^^^^^^^^^^^^^

Constraints:

* MySQL meta-data (i.e. accounts, permissions and Qserv tables structure) has to be aside MySQL data for LSST,
* existing MySQL data for LSST must remains untouched if MySQL meta-data changes.

Currently MySQL meta-data is generated inside the Docker configured images using Qserv configuration procedure. This must evolve in order to ease co-location of MySQL meta-data and data for LSST in the same directory of host machine.

User id mapping
^^^^^^^^^^^^^^^

Currently *qserv* user id in the container has to be equal to user id which own MySQL data directory on the host machine.
Additional work is required to study if Docker latest versions provide user mapping feature which would remove this constraint.

Orchestration
-------------

* **Implement Swarm.** Please note that Docker Swarm feature is not stable yet: https://docs.docker.com/engine/swarm/key-concepts, Swarm API has changed a lot with the newest Docker `v1.12.0-rc1`.
  However, current Swarm proof of concept for Qserv (see https://jira.lsstcorp.org/browse/DM-5967) has been made with previous Swarm API. Nevertheless, this Openstack-based proof of concept is very flexible and designed to easily test cutting-edge container orchestration techniques. 
  
* **Study if Swarm can be used with Compose**, in order to simplify container deployment shell scripts.

Security
^^^^^^^^

* Swarm current version requires SSL certificates on all nodes and system firewall should also be enabled, in order to forbid non authenticated remote access to containers.
  See https://docs.docker.com/swarm/configure-tls/. **It seems SSL certificates are no more useful with latest Swarm version v1.12.0-rc1**
* Deploy trusted containers:

  - Understand Docker Content Trust: https://docs.docker.com/engine/security/trust/content_trust/
  - Eventually set up an LSST Docker trusted registry: https://docs.docker.com/docker-trusted-registry/overview/

Consul
^^^^^^

Prior to Docker `v1.12.0-rc1`, Docker Swarm discovery was requiring use of Consul for production setup: https://docs.docker.com/swarm/discovery/
Nevertheless, it seems that the newest Docker Swarm release candidate does no more require Docker discovery, **so this step should no more be required with latest Swarm versions**.

Micro-services
--------------

Micro-services goal is to add modularity to large applications by splitting them in smaller parts.
**Moving Qserv design towards micro-services require a huge refactoring work.**

Known issues
^^^^^^^^^^^^

* **Monolithic Qserv packages**: Qserv master and worker code are entangled and must
  currently be build and installed simultaneously. 
* **Monolithic configuration procedure**: for both worker and master configurations
* **Monolithic stack**: Qserv build requires multiple external include files,
  and these are only available by installing full eups packages. For example, using MySQL include file currently require a full eups/MySQL install.
* **xrootd uses host machine name and ip stack**, so related container must use host's ip stack.
* **Monolithic xrootd design**: xrootd and cmsd process are strongly tied together, so splitting them in two micro-services is cost-effective.
  Nevertheless, xrootd monolithic design ease a lot deployment and system administration.
* Startup **init.d files launch services in background**, but Docker micro-services philosophy recommend services run in foreground.

Micro-services pros
^^^^^^^^^^^^^^^^^^^

* Log management: each container would produce log files for its embedded Qserv service.
* Update of individual services. Currently, Qserv update impact the whole eups stack.

Micro-services cons
^^^^^^^^^^^^^^^^^^^

* Add complexity in containers management. Current setup only launch one container on each machine, this is simple and easy to manage.
* Worker services are strongly tied to data chunks and must be located on the same machine. So why splitting them?
* Nearly all Master services must be on the same node in order to minimize network use. So why splitting them?
* Two install procedures might be required: one for micro-services embedded inside container and one for regular Qserv installation.
  Currently, the same procedure is used for both regular and container installation.

Tracks
^^^^^^

* In the short-term, MySQL service could run in its own container on all nodes, assuming Qserv has
  been built against a compliant *mysql.h* file.
* In the long-term, micro-services below could also be defined:

  - secondary index, this one is embedded in a MySQL database and can be splitted from Qserv master
  - data loading, this one is not clearly defined yet,
  - data querying, this is most of Qserv code.

Conclusion
^^^^^^^^^^

Current Qserv design is very monolithic and going towards micro-services would be very cost-effective.
Futhermore, added-value of micro-services is not so easy to understand. Indeed, current architecture allows to deploy same binaries on all nodes and to run only one container
on each nodes. This eases a lot packaging, compatibility check, and container orchestration.
Moreover:

* on the worker side, containers are strongly tied to data and MySQL unix socket, so each node has to run all workers services. This constraint reduce drastically micro-services flexibility,
* Versions of multiple Qserv services are highly dependent, and *eups* build performs all compatibility check. Nevertheless, splitting Qserv into fine-grained micro-services would requires an additional mechanism for compatibility check at runtime.

So **splitting all Qserv services in containerized micro-services requires a huge refactoring** and advantages for Qserv architecture are not clear.

**Nevertheless, in the short-term, MySQL could be embedded in a micro-service on both worker and master side.** Use of MySQL system-dependency would reduce Qserv containers size, database configuration and management would be more modular, and current Qserv installation procedure could be improved at moderate cost to support this setup.

****

Copyright 2016 AURA/LSST

This work is licensed under the Creative Commons Attribution 4.0 International License. To view a copy of this license, visit http://creativecommons.org/licenses/by/4.0/.
