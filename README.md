# my dockerfiles

This repository contains scripts to setup a devops environment based on freeipa and docker. It's target to a small cluster of computers, though a single node should work fine. The env contains a management node and a number of computing nodes. The management node also works as a computer node in a single node env. 
- The management node:
  provides services of networking/storage, user authertication, job-dispatch and etc.  -tbd-Failover redundance. 
- The computing nodes 
  provide the computing power based on docker/podman.

In the below is the key software/technology being used, and ubuntu 20.04 is the bassOS on both the management node and computing nodes.
- linux/lvm2/nfs
- docker-compose for freeipa/mysql/gitea/jenkins
- dnsmasq w/pxe to bootup computing nodes -tbd-
- podman/docker/vagrant images/boxes to run on computing nodes

dockerfile: 
- centosDockerfile
  user terminal service

- freeipaDockerfile:   
  directory service (I got its dns conflict with ubuntu host, thus dns is not in docker but using dnsmasq in host.
  see: https://github.com/freeipa/freeipa-container
  
- mysqlDockerfile
  mysql see: https://github.com/docker-library/mysql/tree/master/8.0

- giteaDockerfile
  gitea see: https://docs.gitea.io/en-us/install-with-docker/
  gitea backup/restore see: https://gist.github.com/sinbad/4bb771b916fa8facaf340af3fc49ee43

- podman & skopeo
- podman support rootless container. it specifies the image location as below, (location must be host disk).
  ```toml
    [storage]
    driver = "overlay"
    runroot = "/mnt/backup/"
    graphroot = "/mnt/backup/"
    rootless_storage_path = "/mnt/backup/"
  ````
- skopeo can used to inspect the image, manipunate its storage
  
- dnsmasq:
  tbd https://stackoverflow.com/questions/38816077/run-dnsmasq-as-dhcp-server-from-inside-a-docker-container

host setup,
1. docker install w/ docker-volumn plugin
2. podman
3. misc tools

client setup,
a client can launch the pipeline job remotely with podman_remote, it support attach/detatch (??), so that client can disconnect after job launching. 
If podman is run by a user enabled to run docker, the pipeline task can even create another docker container to run on. 
Another way is submit the task to jenkins, who can create container and deliver jobs to docker engines. (similar setting with podman)
- podman commandline template (tbd)
  ```sh
    docker run -d \
   -v <your_jenkins_home_in_host>:/var/jenkins_home \
   -v /var/run/docker.sock:/var/run/docker.sock \
   -v /usr/local/bin/docker:/bin/docker \
   -p 8080:8080 -p 50000:50000 \
   --name <my_jenkins> jenkins/jenkins:lts
 ```

- run pipeline from podman_remote, which can 
