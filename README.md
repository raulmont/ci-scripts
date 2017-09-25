# CI Scripts

This repository contains scripts and Docker Compose files that form
the Wind River Linux Continuous Integration Prototype.

## Introduction

The prototype has four components:

1) Jenkins Master Docker image
2) Jenkins Agent Docker image with Swarm plugin
3) Ubuntu 16.04 image with all required host packages necessary to
build Yocto.
4) This repo with scripts to orchestrate all the components.

## Getting Started

### Requirements:

make
python3: >= 3.3

Docker CE: >= 17.03

https://docs.docker.com/engine/installation/

Docker Compose: >= 1.13.0

https://docs.docker.com/compose/install/

### Docker Socket permissions

In order for Jenkins Agent to start docker containers without opening
the socket to every user on the machine, the host system needs to
allow a process with uid 1000 rw access to /var/run/docker.sock. This
can be done by adding uid 1000 to the docker group. Currently if the
host docker group has guid 995 to 999, this will enable access.

If the /var/run/docker.sock does not sufficient permissions, the swarm
client image will attempt to enable world rw permissions on the socket
before attempting the build.

### Starting Jenkins

To start Jenkins Master and Agent on a single system using
docker-compose run:

    ./start-jenkins.sh

This will download the images from the Docker Cloud/Hub and start the
images using docker-compose.

The jenkins web UI is accessible at https://localhost/. If attempting
to access the web UI from a different machine, replace localhost with
the name or IP of the server where the repository was cloned to.

The Jenkins interface is behind an nginx reverse proxy which uses a
self signed certificate to provide TLS. Bypassing the insecure web
page warning from the browser will be necessary.

### Scheduling WRLinux Builds

On the same or a different machine, clone this repository. To install
the python-jenkins package locally run:

    make setup
    .venv/bin/python3 ./oe_jenkins_build.py \
        --jenkins https://<jenkins> --configs_file combos-WRLINUX_9_BASE.yaml \
        --configs <config name from combos>

This will contact the Jenkins Master and schedule a build on the
Jenkins Agent.

The combos-WRLINUX_9_BASE.yaml file is a generated list of valid
combinations of qemu bsps and configuration options.

### Scheduling Poky Builds

An example config is provided to demonstrate building Poky locally:

    .venv/bin/python3 ./oe_jenkins_build.py --jenkins <jenkins> \
       --configs_file combos-pyro.yaml --configs=pyro-minimal

To reuse the ubuntu1604_64 image, the poky build uses the WRLinux
buildtools from Github.

### Toaster Integration

All builds enable toaster by default and the prototype uses
Registrator and Consul to discover toaster instances. The toaster
aggregator webapp provides an overview of running toaster instances
and the current progress of each as well as links to the individual
toaster instances.

The toaster aggregator web UI is available at
https://localhost/toaster_aggregator

### Post build operations

The conventional way to add post build operations on Jenkins Pipeline
projects is to add stages to the pipeline. Each pipeline stage would
also require more build parameters be added to the job
configuration. Each additional post build step would require
significant changes and would not compose well.

The post build operations can be run in a different container than the
build to avoid having to add tools like rsync to the build
container. This also allows the build to run without network
access. To select a different post build image:

    .venv/bin/python3 ./oe_jenkins_build.py \
        --jenkins https://<jenkins> --postprocess_image <image>

The prototype contains a generic post build step that does not require
modifications to job config or Jenkinsfile. The post build scripts are
located in the scripts directory and can be selected to run using the
command line:

    .venv/bin/python3 ./oe_jenkins_build.py \
        --jenkins https://<jenkins> --post_success=rsync,cleanup \
        --post_fail=send_email,cleanup

This would run the scripts/rsync.sh and scripts/cleanup.sh scripts
after a successful build and scripts/send_email.sh and
scripts/cleanup.sh after a failed build.

To pass parameters to the post build scripts use:

    .venv/bin/python3 ./oe_jenkins_build.py \
        --jenkins https://<jenkins> --postprocess_args FOO1=bar,FOO2=baz

The prototype will take the parameters, split them and inject them
into the postbuild environment.

To add post build steps, add a script to the scripts directory and add
the script name to the --post_success and/or --post_fail and add any
required parameters to --postprocess_args.

### Developer Builds

An ideal Continuous Integration workflow supports the testing of
patches before they are committed to the "trunk" branches. The
standard git workflow is to use topic branches and have the CI system
run tests against the topic branch.

Yocto projects contain a hierarchy of git repositories and there isn't
a standardized way to create a project area. The wr-lx-setup [1]
attempts to standardize creation of Yocto projects using the
Layerindex as a source of available layers and locations. This
simplifies and standardizes the setup of Yocto projects, but it adds
the Layerindex as a component required by the CI workflow.

The prototype supports creation of a temporary per build Layerindex
and modifying the attributes of a layer in the temporary
Layerindex. This enables a developer to create a topic branch on a
layer git repository and run builds and tests using this branch.

An example workflow:

    .venv/bin/python3 ./oe_jenkins_build.py \
        --jenkins https://<jenkins --configs_file combos-master.yaml
        --configs master-minimal --devbuild_layer_name openembedded-core \
        --devbuild_layer_vcs_url git://github.com/kscherer/openembedded-core.git \
        --devbuild_actual_branch devbuild

The sequence of events is:

1. Temporary Layerindex is created and retrieves master branch info
   from official layers.openembedded.org Layerindex.
2. The vcs_url and actual_branch for the openembedded-core layer in
   temp layerindex is changed and the temp layerindex runs update.py
   to parse this layer. If the update fails, the build also
   fails. This provides a mechanism to test the addition of new
   layers.
3. The wr-lx-setup.sh program is run using the temp layerindex as its
   source and creates a build area using the openembedded layer as
   defined in the supplied vcs_url.
4. After setup is complete, the normal build process continues.
5. After build is complete, the temp layerindex is shutdown and
   cleaned up.

Current Limitations:

1. The Layerindex assumes that bitbake and openembedded-core
   repositories are located on the same git server at the same path.
2. Only changing a single layer is currently supported. There is no
   technical reason why multiple layers could not be changed.
3. For efficiency reasons, the layerindex cache is a shared local
   docker volume and this could cause the update to fail due to a
   timeout if multiple developer builds on the same machine attempt to
   update there own temp layerindex at the same time.

[1]: https://github.com/Wind-River/wr-lx-setup

## Modifying docker images

The CI prototype uses the following images:

- windriver/jenkins-master
- windriver/jenkins-swarm-client
- windriver/ubuntu1604_64
- windriver/toaster_aggregator
- blacklabelops/nginx
- gliderlabs/registrator
- consul
- windriver/layerindex

To test image modifications rebuild the container locally and run:

    ./start-jenkins.sh --no-pull

## TODO

- Build notifications

## Contributing

Contributions submitted must be signed off under the terms of the Linux
Foundation Developer's Certificate of Origin version 1.1. Please refer to:
   https://developercertificate.org

To submit a patch:

- Open a Pull Request on the GitHub project
- Optionally create a GitHub Issue describing the issue addressed by the patch

# Docker Images

This repository contains only the Dockerfiles used to generate the
images. The images are assembled and hosted by Docker on the Docker
Cloud/Hub.

The images contain Open Source software as distributed by the
following projects.

- Ubuntu Linux: https://www.ubuntu.com/
- Alpine Linux: https://www.alpinelinux.org/
- Jenkins: https://jenkins.io/
- Jenkins Plugins: https://plugins.jenkins.io/
- Docker: https://www.docker.com/

# Included Third Party Components

## MIT License

- jQuery Compat JavaScript Library v3.0.0-pre 81b6e46522d8c680f6c38d5e95c732b2b47130b9
- Skeleton V2.0.4
- DataTables 1.10.13

# License

MIT License

Copyright (c) 2017 Wind River Systems Inc.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
