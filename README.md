# What is Singularity

Singularity is a container platform. It allows you to create and run containers that package up pieces of software, just like docker, but more suitable in HPC clusters environment in a way that is portable and reproducible.

# What is the main difference between Singularity and Docker

The use-cases are somewhat different in the default mode between Singularity and Docker. For example, Docker is built and designed to run **root owned services**, while Singularity is designed to run **user owned applications**. You can think about it as using the right tool for the job. Some services will work better on Docker, and apps would probably run better on Singularity. Docker allows those users to have privileged access and control over the host as well so there is a potential security risk there. 

# Why do we need Singularity

Currently we are facing below 2 challenge in docker environment

## Challenge 1: A regular user cannot run docker without being joined in docker group

In order to run docker properly, the user logging in our VMs has to be part of docker group.  However, there is no easy way to add all new users to the group automatically whenever a new VM is created. We need a solution that a regular user can run the container without any elevated access.

### Conclusion

Singularity container can be run by any regular user without being joined in any particular group, as long as the user has the proper permission to access all resource the container is touching. The below I am just using a regular user to run a docker image form docker hub.
```
$ id
uid=1002(tester) gid=1002(tester) groups=1002(tester)

$ singularity run docker://busybox
INFO:    Converting OCI blobs to SIF format
INFO:    Starting build...
Getting image source signatures
Copying blob 92f8b3f0730f done  
Copying config 9dcbd1eb45 done  
Writing manifest to image destination
Storing signatures
2021/05/31 11:06:54  info unpack layer: sha256:92f8b3f0730fef84ba9825b3af6ad90de454c4c77cde732208cf84ff7dd41208
INFO:    Creating SIF file...
Singularity> 
```
 
## Challenge 2: The replacement has to be able to use the Docker images which were created by the users

Singulariy has to be able to use the existing docker image if needed

### Conclusion
Singularity can use docker image, either from docker hub or docker local image repository, without can issue
```
$ singularity run docker://busybox
INFO:    Converting OCI blobs to SIF format
INFO:    Starting build...
Getting image source signatures
Copying blob 92f8b3f0730f done  
Copying config 9dcbd1eb45 done  
Writing manifest to image destination
Storing signatures
2021/05/31 11:06:54  info unpack layer: sha256:92f8b3f0730fef84ba9825b3af6ad90de454c4c77cde732208cf84ff7dd41208
INFO:    Creating SIF file...
Singularity> 
```
 
# Notes for New Singularity Users From Docker End

For most HPC and application specific use-cases, Singularity will work exactly as expected, and probably easier than Docker, but on service based workloads, it might not work as well (but Singularity "instances" will help here). Basically, Docker is built and designed to run root owned services, while Singularity is designed to run user owned applications. 

The below are some scenarios that you need sudo (or docker group) privilege in order to make your Singularity container works:

1. Build from images cached locally by Docker. 
2. When docker-daemon is the bootstrap agent in a Singularity definition file, the sudo requirement is required. 
3. If the Docker container is built with content in root’s home (/root) or needing a root permission
4. If the application needs service daemon or needs to touch any system-level files which needs privileges, such as nginx

> Note: These scenarios are required elevated access as the Docker daemon executes as root. However, if the user issuing the build command is a member of the docker Linux group, then sudo need not be prepended.

# Issues Encountered During the Use

1. It may run out of disk space when running large image app, for example `freesurfer/freesurfer:7.1.1. `

Solution: Change the default location of the cache folder and the temporary folder by setting `SINGULARITY_CACHEDIR` and  `SINGULARITY_TMPDIR` two variables respectively to the new place which has more space. The below are the steps I will suggest and ansible playbook template:

```
1. Modify sssd.conf and change the AD user home folder path to the larger size hard drive folder
$ vim /etc/sssd/sssd.conf
...
[domain/indoc.local]
fallback_homedir = /path/to/secondaryHardDrive/%u@%d
...

2. Import below variable to system environment profile to make the Singularity temporary files save within user's home directory
$ sudo vim /etc/bash.bashrc
...
export SINGULARITY_TMPDIR=$HOME
export SINGULARITY_CACHEDIR=$HOME

3. Sync the user's old home data to the new home folder, if the user has logged into the system below
username=`basename $HOME`
rsync -a /home/${username} $HOME


##########################################
#    Ansible Playbook
##########################################
# Note: The inventory file has to set "home_dir=/path/to/secondaryHardDrive"
---
- name: Singularity installation
  hosts: all
  become: yes
  tasks:
  - name: apt-get update
    apt:
      update_cache: yes

  - name: Install system dependencies
    apt:
      name:
      - build-essential
      - libseccomp-dev 
      - pkg-config 
      - squashfs-tools 
      - cryptsetup
      - git
      - curl

  - name: Install Golang and Singularity
    shell: |
      export VERSION=1.15.8 OS=linux ARCH=amd64 
      wget -O /tmp/go${VERSION}.${OS}-${ARCH}.tar.gz https://dl.google.com/go/go${VERSION}.${OS}-${ARCH}.tar.gz && \
      sudo tar -C /usr/local -xzf /tmp/go${VERSION}.${OS}-${ARCH}.tar.gz
      export GOPATH=${HOME}/go
      export PATH=/usr/local/go/bin:${PATH}:${GOPATH}/bin
      echo 'export GOPATH=${HOME}/go' >> ~/.bashrc && \
      echo 'export PATH=/usr/local/go/bin:${PATH}:${GOPATH}/bin' >> ~/.bashrc 

      # Install golangci-lint
      curl -sfL https://install.goreleaser.com/github.com/golangci/golangci-lint.sh | sh -s -- -b $(go env GOPATH)/bin v1.21.0

      # Clone the repo
      mkdir -p ${GOPATH}/src/github.com/hpcng && \
      cd ${GOPATH}/src/github.com/hpcng && \
      git clone https://github.com/hpcng/singularity.git && \
      cd singularity

      #  Compiling Singularity
      cd ${GOPATH}/src/github.com/hpcng/singularity && \
      ./mconfig && \
      cd ./builddir && \
      make && \
      sudo make install
      singularity version

  - name: Change the user home directory path in sssd.conf
    replace:
      path: /etc/sssd/sssd.conf
      regexp: '^fallback_homedir.*$'
      replace: 'fallback_homedir = {{ home_dir }}/%u@%d'      

  - name: Restart sssd service
    service:
      name: sssd
      state: restarted

  - name: Import the environment variable for Singularity cache path and temporary path which are generated during the build, and sync the user's old home folder data to the new home folder
    lineinfile:
      path: /etc/bash.bashrc
      line: |
        export SINGULARITY_TMPDIR=$HOME
        export SINGULARITY_CACHEDIR=$HOME
        export username=`basename $HOME`
        # Exclude the local users which are not configured via sssd.conf and don't need to sync home direcory
        [ -d /home/${username} ] && [ -d {{ home_dir }}/${username} ] && rsync -a /home/${username}/  $HOME
```

2. NFS filesystem may not support Singularity very well. It will show some warning during  the pull/build. (Same issue in this [ticket](https://groups.google.com/a/lbl.gov/g/singularity/c/p_EfRBrRvI0) regarding NFS share)

Solution: Use secondary hard drive instead.

     
# Pros & Cons
## Pros

1. Customized for HPC clusters scenario, which has been used largely in engineering work-flows for research computing, just like what some hospitals do.
2. Easier to be packaged. The container is saved as a single file which is portable and reproducible. The single file SIF container format is easy to transport and share. （Singularity-CRI makes use of SIF images, which allows you to use all SIF benefits out of the box.）
3. The image is compressed to a fair small size
4. Can inherit the UID from the user who run it directly and can run the image without entering into it, as simple as running a local command line. You are the same user inside a container as outside, and cannot gain additional privilege on the host system by default.
5. Singularity runs container processes without a daemon. They just run as child processes, which means it can effectively run as the running user and doesn’t result in elevated access. （Singularity is known for its security and performance, especially when it comes to HPC. Unlike other popular runtimes, Singularity is not run as a daemon on a node, which prevents lots of security leaks.）
6. Singularity work completely independent of Docker. 
7. It does have ability to import docker images, convert them to singularity images, or run docker container directly
8. Shorter learning curve if you have docker experience
9. Singularity is aimed at compute, that is why Singularity-CRI has built-in NVIDIA GPU support. With it, your Kubernetes cluster won’t need any additional tuning to use GPUs. You use Kubernetes as usual and Singularity-CRI handles the rest.

## Cons
1. Doesn’t support isolation as Docker does.
2. No PID namespace
3. No as popular as Docker so it is lack of community support (but Singularity Slack group is awesome and very supportive!)


Lastly, here some points to consider when evaluating both Docker and Singularity for where to run your computing tasks provided by **@Pedro Alves Batista** :

1. What kind of isolation demands ?
2. Does need much interaction with other network hosts ?
3. Is there specific access policies to resources ?
4. How critical are the data, if any leaking does occur ?
5. How does the orchestration process fit in the solution ?
6. How much time are the team to "switch gears" able without some time of loss on productivity and services downtime ?
7. How can the team evaluate how much computing resources are necessary for running task intensive jobs ?
8. How is the research team aiming to accomplish the results in what concerns HPC workloads and for how long ?


# Reference

https://singularity.hpcng.org/user-docs/master/
**Keyword**: reproducible, portable, security,

https://pythonspeed.com/articles/containers-filesystem-data-processing/
**Keyword**: Volume, working directory, UID

https://www.reddit.com/r/docker/comments/7y2yp2/why_is_singularity_used_as_opposed_to_docker_in/
**Keyword**: Security, scheduling

https://tin6150.github.io/psg/blogger_container_hpc.html
**Keyword**: Comparison

Thanks **@gmk** and **@Pedro Alves Batista** from Singularity Slack community for the great support!!
