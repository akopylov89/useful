# useful
Install and configure docker as slaves server for Jenkins.

    First step will be a docker installation on fse07:

yum install docker

    Modify docker system configuration file to enable docker-api. Default location for config is '/etc/sysconfig/docker':

OPTIONS="-H tcp://0.0.0.0:4243 -H unix:///var/run/docker.sock"

Default location for storage in docker is /var/lib/docker

    Enable docker in systemd:

systemctl enable docker

    Start service:

systemctl start docker

    Get images from docker server:

docker pull centos:latest
docker pull centos:centos6

    Prepare docker files to build our custom images:

mkdir -p /var/vmm/docker/templates/centos{6,7}
touch /var/vmm/docker/templates/centos6/Dockerfile
touch /var/vmm/docker/templates/centos7/Dockerfile

    GS-ROBOT - docker file based on centos6 for testing process

# Default template for gs-jenkins-robot jenkins slave
#
# VERSION               0.1

FROM centos:centos6
MAINTAINER Arthur Novik <anovik@ddn.com>

# Enable EPEL
RUN rpm -Uvh http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm

#Install required packages
RUN yum -y update
RUN yum -y install vim mysql-devel gcc libffi-devel rsync wget tar curl java-1.7.0-openjdk-devel readline-devel git sshfs
RUN yum -y install python-argparse pexpect python-devel python-setuptools
RUN easy_install pip
RUN pip install pyral requests MySQL-python robotframework PyYAML robotframework-sshlibrary robotframework-requests pyopenssl ndg-httpsclient pyasn1
RUN pip isntall jenkinsapi==0.2.22
RUN pip install distribute pbr unittest2 linecache2
RUN easy_install http://ereo-debian.datadirect.datadirectnet.com/Projects/Janus/ALTAV/Release/2.3.0.1.22555/DDN_SFA_API-2.3.0.1.g_r22555-py2.6.egg

#Install idracadm
RUN wget -q -O - http://linux.dell.com/repo/hardware/latest/bootstrap.cgi | bash
RUN yum -y install srvadmin-idracadm ipmitool openssl openssl-devel

#Prepare SSH
RUN yum -y install rsyslog openssh-server openssh-clients screen passwd sudo
RUN sed 's/UsePAM yes/UsePAM no/' -i /etc/ssh/sshd_config
RUN sed 's/#PermitRootLogin yes/PermitRootLogin yes/' -i /etc/ssh/sshd_config
RUN sed 's/#PermitEmptyPasswords no/PermitEmptyPasswords no/' -i /etc/ssh/sshd_config

# Preconfigure for sshd
RUN mkdir /var/run/sshd 
RUN chown root.root /var/run/sshd
RUN chmod 755 /var/run/sshd

# Adding public key
RUN useradd -m gs-jenkins
RUN mkdir /home/gs-jenkins/.ssh/
ADD gs-jenkins.pub /home/gs-jenkins/.ssh/authorized_keys
ADD gs-jenkins /home/gs-jenkins/.ssh/id_rsa
ADD gs-jenkins.pub /home/gs-jenkins/.ssh/id_rsa.pub
RUN chmod 700 /home/gs-jenkins/.ssh/ 
RUN chmod 600 /home/gs-jenkins/.ssh/authorized_keys
RUN chmod 600 /home/gs-jenkins/.ssh/id_rsa
RUN chmod 644 /home/gs-jenkins/.ssh/id_rsa.pub
RUN echo -e 'StrictHostKeyChecking no' > /home/gs-jenkins/.ssh/config
RUN chmod 400 /home/gs-jenkins/.ssh/config
RUN chown -R gs-jenkins.gs-jenkins /home/gs-jenkins/.ssh/
RUN echo 'root:<super-secret-here>' | chpasswd

# SSH login fix. Otherwise user is kicked off after login | No need for centos6
#RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd

RUN sed 's/Defaults *requiretty/#Defaults requiretty/' -i /etc/sudoers
RUN echo 'gs-jenkins ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
RUN echo 'gs-jenkins:<super-secret-here>' | chpasswd

RUN chgrp fuse /dev/fuse
RUN chmod a+rw /dev/fuse
RUN gpasswd -a gs-jenkins fuse

RUN rm -rf /etc/localtime
RUN ln -s /usr/share/zoneinfo/America/Los_Angeles /etc/localtime

EXPOSE 22
RUN /etc/init.d/sshd restart
RUN chkconfig sshd on

    GS-BUILDER - image for gs building process, based on centos 7

# Default template for gs-jenkins-build  slave
#
# VERSION               1.1

FROM centos:latest
MAINTAINER Arthur Novik <anovik@ddn.com>

# Enable EPEL
RUN rpm -Uvh https://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm 

#Install required packages
RUN yum -y update
RUN yum -y groupinstall "Development Tools"
RUN yum -y install mock rpmbuild yum-utils yumdownloader createrepo mkisofs scons

RUN yum -y install vim mysql-devel gcc libffi-devel rsync wget tar curl java-1.7.0-openjdk-devel readline-devel git sshfs zip unzip
RUN yum -y install pexpect python-devel python-setuptools python-mock python-nose pylint python-coverage
RUN easy_install pip
RUN pip install pyral requests MySQL-python robotframework PyYAML robotframework-sshlibrary robotframework-requests pyopenssl ndg-httpsclient pyasn1
RUN pip install jenkinsapi==0.2.22
RUN pip install distribute pbr unittest2 linecache2
RUN easy_install http://ereo-debian.datadirect.datadirectnet.com/Projects/Janus/ALTAV/Release/2.3.0.1.22555/DDN_SFA_API-2.3.0.1.i_r22555-py2.7.egg

#Install idracadm
RUN yum -y install ipmitool openssl openssl-devel

#Prepare SSH
RUN yum -y install rsyslog openssh-server openssh-clients screen passwd sudo
RUN sed 's/UsePAM yes/UsePAM no/' -i /etc/ssh/sshd_config
RUN sed 's/#PermitRootLogin yes/PermitRootLogin yes/' -i /etc/ssh/sshd_config
RUN sed 's/#PermitEmptyPasswords no/PermitEmptyPasswords no/' -i /etc/ssh/sshd_config

# Preconfigure for sshd
RUN mkdir /var/run/sshd 
RUN chown root.root /var/run/sshd
RUN chmod 755 /var/run/sshd

# Adding public key
RUN groupadd -g 500 gs-jenkins
RUN useradd -m gs-jenkins -u 500 -g 500
RUN mkdir /home/gs-jenkins/.ssh/
ADD gs-jenkins.pub /home/gs-jenkins/.ssh/authorized_keys
ADD gs-jenkins /home/gs-jenkins/.ssh/id_rsa
ADD gs-jenkins.pub /home/gs-jenkins/.ssh/id_rsa.pub
RUN chmod 700 /home/gs-jenkins/.ssh/ 
RUN chmod 600 /home/gs-jenkins/.ssh/authorized_keys
RUN chmod 600 /home/gs-jenkins/.ssh/id_rsa
RUN chmod 644 /home/gs-jenkins/.ssh/id_rsa.pub
RUN echo -e 'StrictHostKeyChecking no' > /home/gs-jenkins/.ssh/config
RUN chmod 400 /home/gs-jenkins/.ssh/config
RUN chown -R gs-jenkins.gs-jenkins /home/gs-jenkins/.ssh/


RUN echo 'root:DDNSolutions4U' | chpasswd

# SSH login fix. Otherwise user is kicked off after login | No need for centos6
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd
RUN /usr/bin/ssh-keygen -A

RUN sed 's/Defaults *requiretty/#Defaults requiretty/' -i /etc/sudoers
RUN echo 'gs-jenkins ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
RUN echo 'gs-jenkins:<super-secret>' | chpasswd

RUN chmod a+rw /dev/fuse
RUN usermod -a -G mock gs-jenkins

RUN rm -rf /etc/localtime
RUN ln -s /usr/share/zoneinfo/America/Los_Angeles /etc/localtime

RUN wget https://certs.godaddy.com/repository/gd_bundle-g2-g1.crt && update-ca-trust enable && cp gd_bundle-g2-g1.crt /etc/pki/ca-trust/source/anchors/ && update-ca-trust extract 

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]

    User gs-jenkins should be equal on all server/slaves. Due to that we need to create a user and group with the same uid/guid on all nodes to prevent a problems with rights.

groupadd -g 500 gs-jenkins
useradd -m -u 500 gs-jenkins

    Add key-pair (gs-jenkins{,.pub}) into each docker template directory

[root@fse07 ~]# ls -liah /var/vmm/docker/templates/centos6
total 14K
202 drwxr-xr-x 2 root       root          6 Apr  6 10:11 .
200 drwxr-xr-x 4 root       root          4 Mar 30 10:40 ..
3234 -rw-r--r-- 1 root       root       2.9K Apr  6 09:59 Dockerfile
220 -rw------- 1 gs-jenkins gs-jenkins 1.7K Mar 30 07:01 gs-jenkins
411 -rw-r--r-- 1 gs-jenkins gs-jenkins  392 Mar 31 11:22 gs-jenkins.pub

    Show existing images in vault

[root@fse07 system]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
centos              centos6             f6808a3e4d9e        6 weeks ago         215.7 MB
centos              6                   f6808a3e4d9e        6 weeks ago         215.7 MB
centos              7                   88f9454e60dd        6 weeks ago         223.9 MB
centos              centos7             88f9454e60dd        6 weeks ago         223.9 MB
centos              latest              88f9454e60dd        6 weeks ago         223.9 MB

    Build your own custom images. We should specify a path to directory with Dockerfile, NOT abspath to Dockerfile:

docker build --rm -t gridscaler/ci:gs-robot6 /var/vmm/docker/templates/centos6/
docker build --rm -t gridscaler/ci:gs-builder /var/vmm/docker/templates/centos7/

–rm - remove all temporary files after build will be done

-t - add a tag to image

    To remove image execute “docker rmi <IMAGE ID>”

[root@fse07 system]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              VIRTUAL SIZE
gridscaler/ci       gs-builder          38ead1008b06        About a minute ago   1.432 GB
<none>              <none>              **01807befc36e**        13 days ago          1.386 GB
gridscaler/ci       gs-robot6           1cc44ff3de39        2 weeks ago          1.167 GB
centos              6                   f6808a3e4d9e        6 weeks ago          215.7 MB
centos              centos6             f6808a3e4d9e        6 weeks ago          215.7 MB
centos              7                   88f9454e60dd        6 weeks ago          223.9 MB
centos              centos7             88f9454e60dd        6 weeks ago          223.9 MB
centos              latest              88f9454e60dd        6 weeks ago          223.9 MB

[root@fse07 system]# docker rmi 01807befc36e

Deleted: 01807befc36e7d000fc2e9bbc5c1a70149374c943fb9783e579e2c3d017d1058
Deleted: 9f894c6c12d7a3cf4db3b7bd04d098e5966c074d38787adbfda2bd571ab47637
Deleted: 39c93c7c05bcd8224765367ff2404ba39afdea16b8c9d9b4d8a31a0c65dfab8d
Deleted: a8f909e7336d0b835ba6027b51317f0f240556fc1455eb48f478fd6701a4033b
Deleted: 44a7b23d42195a39c45f828ca222471a178e9db1d222000a93f7852258bdc4cd

    Create containers from custom images:

docker run -t -i -d -p 49555:22 --hostname=gs-builder0 --name=gs-builder0 --privileged=true -v /var/vmm/docker/workspace:/home/gs-jenkins/workspace gridscaler/ci:gs-builder /usr/sbin/sshd -D
docker run -t -i -d -p 49556:22 --hostname=gs-builder1 --name=gs-builder1 --privileged=true -v /var/vmm/docker/workspace:/home/gs-jenkins/workspace gridscaler/ci:gs-builder /usr/sbin/sshd -D
docker run -t -i -d -p 49557:22 --hostname=gs-builder2 --name=gs-builder2 --privileged=true -v /var/vmm/docker/workspace:/home/gs-jenkins/workspace gridscaler/ci:gs-builder /usr/sbin/sshd -D
docker run -t -i -d -p 49558:22 --hostname=gs-builder3 --name=gs-builder3 --privileged=true -v /var/vmm/docker/workspace:/home/gs-jenkins/workspace gridscaler/ci:gs-builder /usr/sbin/sshd -D
docker run -t -i -d -p 49559:22 --hostname=gs-builder4 --name=gs-builder4 --privileged=true -v /var/vmm/docker/workspace:/home/gs-jenkins/workspace gridscaler/ci:gs-builder /usr/sbin/sshd -D
docker run -t -i -d -p 49560:22 --hostname=gs-builder-multijob --name=gs-builder-multijob --privileged=true -v /var/vmm/docker/workspace:/home/gs-jenkins/workspace gridscaler/ci:gs-builder /usr/sbin/sshd -D

docker run -t -i -d -p 49570:22 --hostname=gs-virt-0 --name=gs-virt-0 --privileged=true -v /var/vmm/docker/gs-virt-0:/home/gs-jenkins/job gridscaler/ci:gs-robot6 /usr/sbin/sshd -D
docker run -t -i -d -p 49571:22 --hostname=gs-virt-1_6_1 --name=gs-virt-1.6.1 --privileged=true -v /var/vmm/docker/gs-virt-1.6.1:/home/gs-jenkins/job gridscaler/ci:gs-robot6 /usr/sbin/sshd -D
docker run -t -i -d -p 49572:22 --hostname=gs-virt-2_1_0 --name=gs-virt-2.1.0 --privileged=true -v /var/vmm/docker/gs-virt-2.1.0:/home/gs-jenkins/job gridscaler/ci:gs-robot6 /usr/sbin/sshd -D
docker run -t -i -d -p 49573:22 --hostname=gs-virt-2_2_0 --name=gs-virt-2.2.0 --privileged=true -v /var/vmm/docker/gs-virt-2.2.0:/home/gs-jenkins/job gridscaler/ci:gs-robot6 /usr/sbin/sshd -D
docker run -t -i -d -p 49574:22 --hostname=gs-virt-2_3_0 --name=gs-virt-2.3.0 --privileged=true -v /var/vmm/docker/gs-virt-2.3.0:/home/gs-jenkins/job gridscaler/ci:gs-robot6 /usr/sbin/sshd -D
docker run -t -i -d -p 49575:22 --hostname=gs-virt-3_0_0 --name=gs-virt-3.0.0 --privileged=true -v /var/vmm/docker/gs-virt-3.0.0:/home/gs-jenkins/job gridscaler/ci:gs-robot6 /usr/sbin/sshd -D
docker run -t -i -d -p 49576:22 --hostname=gs-virt-3_0_1 --name=gs-virt-3.0.1 --privileged=true -v /var/vmm/docker/gs-virt-3.0.1:/home/gs-jenkins/job gridscaler/ci:gs-robot6 /usr/sbin/sshd -D
docker run -t -i -d -p 49586:22 --hostname=gs-virt-3_2_0 --name=gs-virt-3.2.0 --privileged=true -v /var/vmm/docker/gs-virt-3.2.0:/home/gs-jenkins/job gridscaler/ci:gs-robot6 /usr/sbin/sshd -D

Keys:

    t, –tty=true|false When set to true Docker can allocate a pseudo-tty and attach to the standard input of any container. This can be used, for example, to run a throwaway interactive shell. By default False.
    -i, –interactive=true|false When set to true, keep stdin open even if not attached. The default is false.
    -d, –detach=true|false Detached mode. This runs the container in the background.
    -p, –publish=[] Publish a container's port to the host (format: ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort | containerPort) (use docker port to see the actual map-ping)
    -v, –volume=volume[:ro|:rw] Bind mount a volume to the container. By default, the volumes are mounted read-write.

    Show all containers:

[root@fse07 system]# docker ps -a
CONTAINER ID        IMAGE                      COMMAND               CREATED             STATUS              PORTS                   NAMES
4de6fa075e58        gridscaler/ci:gs-builder   "/usr/sbin/sshd -D"   2 hours ago         Up 2 hours          0.0.0.0:49560->22/tcp   gs-builder-multijob   
f75c6830fbe9        gridscaler/ci:gs-builder   "/usr/sbin/sshd -D"   2 hours ago         Up 2 hours          0.0.0.0:49559->22/tcp   gs-builder4           
cbe0a04cf242        gridscaler/ci:gs-builder   "/usr/sbin/sshd -D"   2 hours ago         Up 2 hours          0.0.0.0:49557->22/tcp   gs-builder2           
51a017ca7d4b        gridscaler/ci:gs-builder   "/usr/sbin/sshd -D"   2 hours ago         Up 2 hours          0.0.0.0:49555->22/tcp   gs-builder0           
20d61cb11b7a        gridscaler/ci:gs-builder   "/usr/sbin/sshd -D"   2 hours ago         Up 2 hours          0.0.0.0:49556->22/tcp   gs-builder1           
650578fc8d5f        gridscaler/ci:gs-builder   "/usr/sbin/sshd -D"   2 hours ago         Up 2 hours          0.0.0.0:49558->22/tcp   gs-builder3           
e097f8858b51        gridscaler/ci:gs-robot6    "/usr/sbin/sshd -D"   9 days ago          Up 9 days           0.0.0.0:49583->22/tcp   gs-block1             
3e7cf3cf213d        gridscaler/ci:gs-robot6    "/usr/sbin/sshd -D"   9 days ago          Up 9 days           0.0.0.0:49582->22/tcp   gs-block0         

    Remove container with:

docker rm <CONTAINER_ID or NAME>

    To delete all containers

docker rm `docker ps -a`

    Show current configuration for existing image or container:

[root@fse07 system]# docker inspect gridscaler/ci:gs-robot6
[{
    "Architecture": "amd64",
    "Author": "Arthur Novik \u003canovik@ddn.com\u003e",
    "Comment": "",
    "Config": {
        "AttachStderr": false,
        "AttachStdin": false,
        "AttachStdout": false,
        "Cmd": null,
        "CpuShares": 0,
        "Cpuset": "",
        "Domainname": "",
        "Entrypoint": null,
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ],
        "ExposedPorts": {
            "22/tcp": {}
        },
        "Hostname": "cd2c5e92b2a0",
        "Image": "cbd09b072899ac0ce85c5f4cfdc4ffe73bc0635705f12738b0953e12862d93a3",
        "Memory": 0,
        "MemorySwap": 0,
        "NetworkDisabled": false,
        "OnBuild": [],
        "OpenStdin": false,
        "PortSpecs": null,
        "StdinOnce": false,
        "Tty": false,
        "User": "",
        "Volumes": null,
        "WorkingDir": ""
    },
    "Container": "cf6cfbab619eb6fcf9443be79a924ba0a474430125ec436e15dda811123c4810",
    "ContainerConfig": {
        "AttachStderr": false,
        "AttachStdin": false,
        "AttachStdout": false,
        "Cmd": [
            "/bin/sh",
            "-c",
            "chkconfig sshd on"
        ],
        "CpuShares": 0,
        "Cpuset": "",
        "Domainname": "",
        "Entrypoint": null,
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ],
        "ExposedPorts": {
            "22/tcp": {}
        },
        "Hostname": "cd2c5e92b2a0",
        "Image": "cbd09b072899ac0ce85c5f4cfdc4ffe73bc0635705f12738b0953e12862d93a3",
        "Memory": 0,
        "MemorySwap": 0,
        "NetworkDisabled": false,
        "OnBuild": [],
        "OpenStdin": false,
        "PortSpecs": null,
        "StdinOnce": false,
        "Tty": false,
        "User": "",
        "Volumes": null,
        "WorkingDir": ""
    },
    "Created": "2015-03-31T17:03:50.326208203Z",
    "DockerVersion": "1.3.2",
    "Id": "1cc44ff3de396f1ef5590209ff8fabe1e4fd345039bda44bd7503cb1566bd83b",
    "Os": "linux",
    "Parent": "cbd09b072899ac0ce85c5f4cfdc4ffe73bc0635705f12738b0953e12862d93a3",
    "Size": 98,
    "VirtualSize": 1167425644
}]  

[root@fse07 system]# docker inspect --format='{{.Volumes}}'  gs-builder0
map[/home/gs-jenkins/workspace:/var/vmm/docker/workspace]

    Add containers to systemd, it will allow to launch them automatically.

/usr/lib/systemd/system/gs-builder0.service

[Unit]
Description=GridScaler build container
Author=Arthur Novik
After=docker.service
Requires=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker start -a gs-builder0
ExecStop=/usr/bin/docker stop -t 2 gs-builder0

[Install]
WantedBy=multi-user.target

    Updating image from existing container

Syntax:

docker commit -m 'checkpatch' <container_ID> <image:tag>

[root@fse07 templates]# docker stop builder
[root@fse07 templates]# docker ps -a
CONTAINER ID        IMAGE                      COMMAND               CREATED             STATUS                     PORTS                   NAMES
758c8fddac6b        gridscaler/ci:gs-robot6    "/usr/sbin/sshd -D"   7 days ago          Up 7 days                  0.0.0.0:49600->22/tcp   gs-ccm                
5d138f5db6d3        gridscaler/ci:gs-robot6    "/usr/sbin/sshd -D"   12 days ago         Up 12 days                 0.0.0.0:49577->22/tcp   gs-virt-3.1.0         
d92b615b5f18        gridscaler/ci:builder      "/usr/sbin/sshd -D"   2 weeks ago         Exited (0) 2 minutes ago                           builder               
7d44095661f5        655adf2f14ff               "/usr/sbin/sshd -D"   4 weeks ago         Up 3 weeks                 0.0.0.0:49584->22/tcp   reposyncer  

[root@fse07 templates]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              VIRTUAL SIZE
<none>              <none>              03fc2c80acfa        About a minute ago   2.134 GB
gridscaler/ci       builder             1002aa96785e        2 weeks ago          1.981 GB
<none>              <none>              283ff58ae05a        2 weeks ago          1.757 GB
gridscaler/ci       reposyncer          72f310cca9a0        4 weeks ago          403.4 MB
rhel7/rhel          7.1-6               65de4a13fc7c        10 weeks ago         154.9 MB
rhel7/rhel          latest              65de4a13fc7c        10 weeks ago         154.9 MB
gridscaler/ci       gs-builder          38ead1008b06        3 months ago         1.432 GB
gridscaler/ci       gs-robot6           1cc44ff3de39        3 months ago         1.167 GB
centos              6                   f6808a3e4d9e        4 months ago         215.7 MB
centos              centos6             f6808a3e4d9e        4 months ago         215.7 MB
centos              latest              88f9454e60dd        4 months ago         223.9 MB
centos              7                   88f9454e60dd        4 months ago         223.9 MB
centos              centos7             88f9454e60dd        4 months ago         223.9 MB

[root@fse07 templates]# docker commit -m 'checkpatch' d92b615b5f18 gridscaler/ci:builder
8f608132db0315338c894e76584bd94eae633ce3741a12250a1935937997e23a

[root@fse07 templates]# docker start builder

    Save/Load images

docker save image_ID_here > /tmp/myimage.tar
docker load < /tmp/myimage.tar

    Docker info

[root@fse07 system.slice]# docker info
Containers: 24
Images: 191
Storage Driver: devicemapper
 Pool Name: docker-253:4-1050453-pool
 Pool Blocksize: 65.54 kB
 Data file: /var/lib/docker/devicemapper/devicemapper/data
 Metadata file: /var/lib/docker/devicemapper/devicemapper/metadata
 Data Space Used: 39.53 GB
 Data Space Total: 107.4 GB
 Metadata Space Used: 31.16 MB
 Metadata Space Total: 2.147 GB
 Library Version: 1.02.84-RHEL7 (2014-03-26)
Execution Driver: native-0.2
Kernel Version: 3.10.0-123.20.1.el7.x86_64
Operating System: CentOS Linux 7 (Core)

from glob import glob
containers_memory = glob('/sys/fs/cgroup/memory/system.slice/docker-*.scope/memory.usage_in_bytes')
Memory_usage_in_MB = []
for container in containers_memory:
  with open(container, 'rb') as file:
    Memory_usage_in_MB.append(int(file.readline().strip())/1024/1024)
Total_usage = sum([x for x in Memory_usage_in_MB])
print Total_usage, "Mb"
