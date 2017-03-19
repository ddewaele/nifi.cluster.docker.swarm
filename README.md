# Apache Nifi Cluster in Docker Swarm

## Installing docker

We're going to be installing `docker-ce-17.03.0.` , this new release schema (using year / month for quarterly releases) replaces the older versioning scheme (`Docker version 1.12.1`)

In order to install Docker execute the following commands:

    yum install -y yum-utils

    yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo

    yum makecache fast
    yum -y install docker-ce-17.03.0.ce-1.el7.centos
    systemctl start docker
    systemctl enable docker

## Firewall changes

### Manager

    firewall-cmd --add-port=22/tcp --permanent
    firewall-cmd --add-port=2376/tcp --permanent
    firewall-cmd --add-port=2377/tcp --permanent
    firewall-cmd --add-port=7946/tcp --permanent
    firewall-cmd --add-port=7946/udp --permanent
    firewall-cmd --add-port=4789/udp --permanent

    firewall-cmd --reload
    systemctl restart docker

### Workers

    firewall-cmd --add-port=22/tcp --permanent
    firewall-cmd --add-port=2376/tcp --permanent
    firewall-cmd --add-port=7946/tcp --permanent
    firewall-cmd --add-port=7946/udp --permanent
    firewall-cmd --add-port=4789/udp --permanent

    firewall-cmd --reload
    systemctl restart docker
