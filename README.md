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

## Niffi cluster

A couple of things to note when running components that have inherent clustering support themselves. In a typical scenario when deploying stateless services into docker swarm, docker swarm itself takes care of replication, load balancing and failover. It is in essesnce an out-of-the-box cluster solution and your services don't need to worry about anything. You take 1 service definition, and you give it a desired state (for example I want to have 2 replicaes).

When it comes to components that have their own clustering solution (like Nifi, but also Zookeeper or Kafka) we need a different approach. In that scenario, if we want to make them available in Docker Swarm, we cannot let Swarm handle the replication for us, but we ned to configure them in such a way that we let the component itself do the clustering.

Nifi has a zero-master clusternig and depends on Apache ZooKeeper to provide automatic election of different clustering-related roles. In Nifi's case we have

- Primary Node
- Cluster Coordinator

### Primary Node

Explain (todo)

### Cluster Coordinator

Explain (todo)

## Nifi Cluster related issus

In Docker swarm, every service instance has 2 ip addresses. 1 internal address specific to that particular instance, and 1 DNS address that is the same for all instances. In the current swarm implementation, the way that a host is resolved differs from the fact if the resolve took place internally (inside the container), or from another place in the swarm (outside the container, but in the same swarm network). This is illustrated in https://github.com/docker/docker/issues/30963 and has some consequences for Apache Nifi.

    docker service inspect   --format='{{json .Endpoint.VirtualIPs}}' stack1_nifi1
    [{"NetworkID":"ubzje45aw1e38b6p17qaxlns5","Addr":"10.255.0.8/16"},{"NetworkID":"r1nhzzbj9tju52zvc448wogdy","Addr":"10.0.0.10/24"}]

Inside the container, when we ping th instance we get the same ip address:

    [root@centos-a ~]# docker exec -ti 215a14b35c82 /bin/sh
    # ping stack1_nifi1
    PING stack1_nifi1 (10.0.0.10): 56 data bytes
    64 bytes from 10.0.0.10: icmp_seq=0 ttl=64 time=0.047 ms
    64 bytes from 10.0.0.10: icmp_seq=1 ttl=64 time=0.074 ms

However, when we look at /etc/hosts, we see a different ip address:

    # cat /etc/hosts
    127.0.0.1 localhost
    ::1 localhost ip6-localhost ip6-loopback
    fe00::0 ip6-localnet
    ff00::0 ip6-mcastprefix
    ff02::1 ip6-allnodes
    ff02::2 ip6-allrouters
    10.0.0.11 215a14b35c82

When Nifi starts, it starts a web server for the ui and therefor needs to bind to a certain ip / host. When nothing is provided, it binds to 0.0.0.0, listens on all interfaces (including 10.0.0.11 and 10.0.0.10).

Because `nifi.web.http.host` is blank it defaults to `localhost` and uses that to identify himself. 
As a consequence, the NodeID that it uses to identify itself in the cluster becomes localhost:8080.

    # Leave blank so that it binds to all possible interfaces 
    nifi.web.http.host= 
    nifi.web.http.port=8080  #(8085 on the other node) 

As Nifi seems to be using `nifi.web.http.host` to generate the NodeID, and to identify the nodes to do UI replication. the obvious fix would be to specify the actual hostname here. That way Nifi would bind to the interfaces configured for that host (by doing a reverse DNS lookup).

The problem with that is that nifi only listens on `10.10.0.10` when using `nifi.web.http.host=nifi1`

When starting with `nifi.web.http.host=localhost` from inside the container:

    curl -v http://10.0.0.10:8080/nifi/         connection refused
    curl -v http://10.0.0.11:8080/nifi/         connection refused
    curl -v http://stack1_nifi1:8080/nifi/      connection refused

When starting with `nifi.web.http.host=nifi1` from inside the container:

    curl -v http://10.0.0.10:8080/nifi/         connection refused
    curl -v http://10.0.0.11:8080/nifi/         connection refused
    curl -v http://stack1_nifi1:8080/nifi/      connection refused

When starting with `nifi.web.http.host=10.0.0.10` from inside the container:

    curl -v http://10.0.0.10:8080/nifi/         connection refused
    curl -v http://10.0.0.11:8080/nifi/         connection refused
    curl -v http://stack1_nifi1:8080/nifi/      connection refused

When starting with `nifi.web.http.host=10.0.0.11` from inside the container:

    curl -v http://10.0.0.10:8080/nifi/         http 200
    curl -v http://10.0.0.11:8080/nifi/         http 200
    curl -v http://stack1_nifi1:8080/nifi/      http 200
    
So we only get good results when the listen address is 10.0.0.11.
However, when specifying `nifi.web.http.host=nifi1`, it seems to listen at 10.0.0.10 resulting in failures.
As we obviously don't want to hardcode ip addresses in web.http.host, and we cannot use `localhost` we're left with using `nifi`.

As a `workaround`, we can specify the `hostname` flag in our docker config, to inject `nifi1` in the hostfile.

That way, when we look in the hosts file, we see that nifi1 will resolve to 10.0.0.11 and all is good.

    [root@centos-a ~]# docker exec -ti f2e /bin/sh
    # cat /etc/hosts
    127.0.0.1	localhost
    ::1	localhost ip6-localhost ip6-loopback
    fe00::0	ip6-localnet
    ff00::0	ip6-mcastprefix
    ff02::1	ip6-allnodes
    ff02::2	ip6-allrouters
    10.0.0.11	nifi1


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
    
