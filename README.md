# my dockerfiles

## overview
This repository contains scripts to setup a devops environment based on freeipa and docker. It's target to a small cluster of computers, though a single node should work fine. The env contains a management node and a number of computing nodes. The management node also works as a computer node in a single node env. 
- The management node:
  provides services of networking/storage, user authertication, job-dispatch and etc. 
- The computing nodes 
  provide the computing power based on docker/podman.

In the below is the key software/technology being used, and ubuntu 20.04 is the bassOS on both the management node and computing nodes.
- linux/lvm2/nfs
- docker-compose for freeipa/mysql/gitea/jenkins
- dnsmasq w/pxe to bootup computing nodes -dropped-
  (for a large cluster, using clearlinux is a better solution which provides: controlled os update/mixer; reference setup of software stack; and certainly PXE) 
- podman/docker/vagrant run on computing nodes

## dockerfiles: 
these dockerfiles below are used to generate local maintained container images, if image with same functionality are available from public docker image registry, those image could be used and retire or simply the local dockerfiles.

- centosDockerfile: 
this container run on terminal server, mount the data-volume, create a vnc for user to connect. it is an alternative if a user dont like the default vnc-server on the terminal server directly.  the user can create container by using podman, and connect into it through vnc. otherwise user could start vnc service on terminal server directly if the default desktop gui works for them.  
  
      ```bash
        docker run --volume /data:/data:rw -v /tmp/.X11-unix:/tmp/.X11-unix --env DISPLAY=unix$DISPLAY -it ubuntu bash
      ```
the terminal servers in intranet are just normal worknode.
the terminal servers are supported to be connected from internet and they are DMZ computers as they have two network interfaces, with one connected with cluster and another open to outside. users from outside of intranet can only connect to DMZ servers which has firewall/port-forwarding enforced e.g.
   - external nic has public ip and domain name
   - vpn connect supported through wireguard to its external nic
   - user connect are forwarded to internal nic e.g. access vnc-server run on cluster which are controlled by iptables.

- freeipaDockerfile: 
provide directory/authentication service (I got its dns conflict with ubuntu host, thus dns is not in docker but using dnsmasq in host.)
it should be only one instance in the pool. see: https://github.com/freeipa/freeipa-container
prefer to use the official image freeipa/freeipa-server:centos-?

- jenkinsDockerfile: 
provide jenkins service; jenkins can use docker daemon at a node thus can distribute workload to cluster. Jenkins also supports distribute workload with ssh (ssh agent).
      ```bash
        docker run -d \
       -v <your_jenkins_home_in_host>:/var/jenkins_home \
       -v /var/run/docker.sock:/var/run/docker.sock \
       -v /usr/local/bin/docker:/bin/docker \
       -p 8080:8080 -p 50000:50000 \
       --name <my_jenkins> jenkins/jenkins:lts
     ```
- mysqlDockerfile
mysql is required by gitea see: https://github.com/docker-library/mysql/tree/master/8.0

- giteaDockerfile
  gitea see: https://docs.gitea.io/en-us/install-with-docker/
  gitea backup/restore see: https://gist.github.com/sinbad/4bb771b916fa8facaf340af3fc49ee43
  also ref https://github.com/go-gitea/gitea/tree/master/docker
  
## podman instead of docker
master node has the pre-installed tools exports to work nodes with nfs. when a user wants extra tool or need different work enviornment (all nodes are ubuntu20.04), they could create rootless container with podman, which supporst rootless container which is non-privilege.  it specifies the image location as below,  and image location must be host disk *instead of nfs*. otherwise container/images are under user's home.
  ```toml
    [storage]
    driver = "overlay"
    runroot = "/mnt/backup/"
    graphroot = "/mnt/backup/"
    rootless_storage_path = "/mnt/backup/"
  ```
- skopeo can used to inspect the image, manipunate its storage
  
## dnsmasq: to support dhcp and pxe and automatically deploy the computing node (-tbd-) 
  (it is desired to wake up the backup server; start it with hosted image e.g. run a backup and log a report; and then power-off/suspend the server).
  keep the below as reference, but not implemented in my experiment. 
  https://stackoverflow.com/questions/38816077/run-dnsmasq-as-dhcp-server-from-inside-a-docker-container
  pxe env build:  https://netboot.xyz/selfhosting/
  pxe selfhost container: https://github.com/linuxserver/docker-netbootxyz
  pxe w/ dhcp: https://github.com/samdbmg/dhcp-netboot.xyz https://github.com/kmanna/docker-nat-router
  
# setup
1. master node setup see: [host_setup.md](host_setup.md)
  - tools installed on host
    - docker install w/ docker-volumn plugin
    - podman
    - site tools 
    - conda
  - docker_compose file
    - docker compose can start as daemon, the generated image has special signiture so that it doesn't rebuild the image each time. 
    
2. compute node setup,
  - docker/podman install w/ nfs volume
  - nextflow 
    local job is dispatched with nextflow pipeline (could use podman rootless) on the node. (or docker swarm to the cluster directly?)
  - compute nodes take workload from jenkins
    
3. work flow
  - user connects to the terminal server
  - user runs their job through nf pipeline at local node, which could creates more container locally or remotely(with podman remote).
  - or user submits their job/pipeline into jenkins which delivers jobs to cluster
  - common dir (nfs exported from master node) for tools / projects data;
