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
    
## Initializing a swarm

The first thing we're going to do is initialize the swarm. This is done by executing the following command :

    sudo docker swarm init

If all goes well a Swarm will be initialized on the machine, where it will be accepting connections from nodes on TCP socket 2377.

    Swarm initialized: current node (z71es0zrd4vonzo2gtdvvslcf) is now a manager.

    To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-57oalkpfsq8o70wxjnjt378068moqma67k3w42lynwkxptpl76-cl1y7jle0byil5cxbc8ld7xs5 \
    192.168.0.191:2377

If you have multiple interfaces, and you want to bind to a particular IP, you can use the `advertise-addr <IP ADDRESS>` option.

### Swarm status

Congratulations, you've created your first Swarm. If you have any issues please refer to the Swarm trouble shooting section below.

You can now ready view the status of your swarm by issueing the following:

    [root@centos-vm ~]# docker node ls
    ID                           HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
    z71es0zrd4vonzo2gtdvvslcf *  centos-vm  Ready   Active        Leader

So now we have 1 Leader in the swarm cluster (ourselves). Now its time to add workers

### Adding worker nodes

As you can see from the output, worker nodes can add the swarm by issuing the command above. Not that you can always see this output again by executing the command below on the manager :

    docker swarm join-token worker

Copy paste the output on a worker node on centos-a, and that worker node will become part of the swarm. If the node was already part of a swarm you'll need to leave the current swarm first.

    [root@centos-vm ~]# docker node ls
    ID                           HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
    uygn54lnlq8ecm6wazh3b8k80    centos-a   Ready   Active        
    z71es0zrd4vonzo2gtdvvslcf *  centos-vm  Ready   Active        Leader

Do the same on centos-b

    [root@centos-vm ~]# docker node ls
    ID                           HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
    77lwga8siyh7yx4m92aqd3mlb    centos-b   Ready   Active        
    uygn54lnlq8ecm6wazh3b8k80    centos-a   Ready   Active        
    z71es0zrd4vonzo2gtdvvslcf *  centos-vm  Ready   Active        Leader

### Starting the swarm

Before we can start the swarm we need to do 2 things

#### Clone this repo in /root/

    git clone https://github.com/ddewaele/nifi.cluster.docker.swarm.git

#### Create mount points on all nodes

    mkdir -p /srv/nifi/flow
    mkdir -p /srv/nifi/content_repository
    mkdir -p /srv/nifi/database_repository
    mkdir -p /srv/nifi/flowfile_repository
    mkdir -p /srv/nifi/provenance_repository
    mkdir -p /srv/nifi/work
    mkdir -p /srv/nifi/logs
    mkdir -p /tmp/data

When this is done we can start the stack like this:

    docker stack deploy --with-registry-auth -c docker-compose.yml stack1

### Stopping the swarm

    docker stack rm  stack1

### Swarm troubleshooting

#### IP address

When you create s swarm, part of the output will be an ip addesss port where workers can join. Double check the IP address when creating a swarm cluster. 
In an environment with dynamic ips, it's possible that swarm remembered the previous IP. If you want to completely re-initialize the cluster you can use the `--force-new-cluster` option.

#### Leave cluster if needed

If the machine is already part of the cluster you'll get the following error:

    Error response from daemon: This node is already part of a swarm. Use "docker swarm leave" to leave this swarm and join another one.

If the manager is not part of a swarm you'll get this:

    Error response from daemon: This node is not a swarm manager. Use "docker swarm init" or "docker swarm join" to connect this node to swarm and try again.

#### Docker live restore

If you get the following error message 

    Error response from daemon: --live-restore daemon configuration is incompatible with swarm mode

You need to edit the following file and turn off live restore.

    cat /etc/docker/daemon.json
    {
        "live-restore": true
    }

#### Check var/log/messages

If you're unable to join the swarm cluster, check your '/var/log/messages' to make sure that the nodes can contact the swarm leader

    Mar 17 02:27:48 localhost dockerd: time="2017-03-17T02:27:48.650544962-04:00" level=error msg="Failed to join memberlist [172.20.10.3] on retry: 1 error(s) occurred:\n\n* Failed to join 172.20.10.3: dial tcp 172.20.10.3:7946: getsockopt: connection refused"
    Mar 17 02:27:49 localhost dockerd: time="2017-03-17T02:27:49.650392919-04:00" level=error msg="Failed to join memberlist [172.20.10.3] on retry: 1 error(s) occurred:\n\n* Failed to join 172.20.10.3: dial tcp 172.20.10.3:7946: getsockopt: connection refused"

#### Firewall issues

Double check that there are no firewall issues by issueing a telnet to the swarm leader

    root@centos-a ~]# telnet 192.168.0.191 2377
    Trying 192.168.0.191...
    Connected to 192.168.0.191.
    Escape character is '^]'.

    [root@centos-a ~]# telnet 192.168.0.191 7946
    Trying 192.168.0.191...
    Connected to 192.168.0.191.
    Escape character is '^]'.

Dobule check that yout swarm leader is listening to the various ports

    [root@centos-vm swarm]# netstat -tulpn | grep docker
    tcp6       0      0 :::7946                 :::*                    LISTEN      1039/dockerd        
    tcp6       0      0 :::2377                 :::*                    LISTEN      1039/dockerd        
    udp6       0      0 :::7946                 :::*                                1039/dockerd        

#### Joining cluster commands

if you forgot how to join the swarm cluster as either a manager or as a worker, issue the following commands 

    docker swarm join-token manager
    docker swarm join-token worker

 
#### Cluster in an consistent state

Sometimes swarm can get confused, and you might want to leave the swarm altogether.

    [root@centosvm multi-user.target.wants]# docker swarm init --advertise-addr 192.168.0.191
    Error response from daemon: This node is already part of a swarm. Use "docker swarm leave" to leave this swarm and join another one.
    [root@centosvm multi-user.target.wants]# docker swarm leave
    Error response from daemon: context deadline exceeded
    [root@centosvm multi-user.target.wants]# docker node ls
    Error response from daemon: rpc error: code = 2 desc = raft: no elected cluster leader

In that case you can force the creation of a new cluster like this 

    docker swarm init --force-new-cluster
    docker swarm init --advertise-addr 172.16.3.154 --force-new-cluster

#### Ingres network not created on worker nodes

Sometimes you see this in `/var/log/messages` 

    Mar  6 15:05:45 localhost dockerd-current: time="2017-03-06T15:05:45.628495856-05:00" level=info msg="{Action=networks, Username=root, LoginUID=0, PID=3581}"
    Mar  6 15:07:01 localhost dockerd-current: time="2017-03-06T15:07:01.848843383-05:00" level=info msg="{Action=leave, Username=root, LoginUID=0, PID=3588}"
    Mar  6 15:07:28 localhost dockerd-current: time="2017-03-06T15:07:28.444636373-05:00" level=info msg="{Action=join, Username=root, LoginUID=0, PID=3592}"
    Mar  6 15:07:28 localhost dockerd-current: time="2017-03-06T15:07:28.447964027-05:00" level=warning msg="Could not find ingress network while leaving: network ingress not found"
    Mar  6 15:07:28 localhost dockerd-current: time="2017-03-06T15:07:28.463132389-05:00" level=info msg="Waiting for TLS certificate to be issued..."
    Mar  6 15:07:28 localhost dockerd-current: time="2017-03-06T15:07:28.469101946-05:00" level=info msg="Downloaded new TLS credentials with role: swarm-worker."
    Mar  6 15:07:28 localhost dockerd-current: time="2017-03-06T15:07:28.547430841-05:00" level=info msg="{Action=01oa7e502xhbsgo7681brbmxn, Username=root, LoginUID=0, PID=3592}"

where we should have seen this

    Mar  6 15:14:45 localhost dockerd-current: time="2017-03-06T15:14:45.923136031-05:00" level=info msg="{Action=join, Username=root, LoginUID=0, PID=2500}"
    Mar  6 15:14:45 localhost dockerd-current: time="2017-03-06T15:14:45.967385587-05:00" level=info msg="Waiting for TLS certificate to be issued..."
    Mar  6 15:14:45 localhost dockerd-current: time="2017-03-06T15:14:45.974734682-05:00" level=info msg="Downloaded new TLS credentials with role: swarm-worker."
    Mar  6 15:14:46 localhost dockerd-current: time="2017-03-06T15:14:46.075697387-05:00" level=info msg="Initializing Libnetwork Agent Listen-Addr=0.0.0.0 Local-addr=192.168.0.122 Adv-addr=192.168.0.122 Remote-addr =192.168.0.191"
    Mar  6 15:14:46 localhost dockerd-current: time="2017-03-06T15:14:46.075744718-05:00" level=info msg="Gossip cluster hostname node2-650c2ff501e4"
    Mar  6 15:14:46 localhost dockerd-current: time="2017-03-06T15:14:46.109140645-05:00" level=info msg="{Action=1n8aitz5d4omfuu3n69m4jt3e, Username=root, LoginUID=0, PID=2500}"
    
