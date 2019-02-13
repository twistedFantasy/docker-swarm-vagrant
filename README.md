# docker-swarm-vagrant

The Goal of this project is to provide a simple example of using Docker Swarm mode with VirtualBox. At the end you will have 2 VirtualBox instances powered by Debian 9
with installed Docker(18.06.1) which will be  a docker cluster with 1 manager and 1 worker, in this cluster we deploy nginx service with 5 replicas
and redis service placed on manager machine in cluster. 

#### Requirements:
On your laptop you should have installed next software:
```
Vagrant 2.0.2 or higher: https://www.vagrantup.com/docs/installation/
VirtualBox 5.2.22 or higher: https://www.virtualbox.org/wiki/Downloads 
```

#### How to run it:
To create initial machines described in Vagrantfile you should navigate to the root of this project and execute next command:
```
vagrant up
```

When this command will finish execution you will have 2 VirtualBox machines, you can check it with next command:
```
vagrant global-status

# output
id       name          provider   state    directory 
-----------------------------------------------------------------------------------------------------------------------------------
3e0aca1  swarm_manager virtualbox running  /home/ubuntu/docker-swarm-vagrant    
baca7ad  swarm_worker  virtualbox running  /home/ubuntu/docker-swarm-vagrant    
 
```
Let's initialize our Docker Swarm cluster, we need to ssh in VirtualBox machine which will have a manager role
```
vagrant ssh swarm_manager
```
To initialize Docker Swarm you need to execute next command:
```
docker swarm init --advertise-addr 192.168.56.101

#output:
Swarm initialized: current node (f7ysphew4er8dkmvefkbot4vc) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-59uvhwxzz34cj5t1kiykwidrzg4afepdz0r96rjk5e1bcxx9tz-dmfs8pqr71ndranwm5xy68of3 192.168.56.101:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
The output will contain a command which we need to execute on any other machine which we want to join to our Docker Swarm Cluster.
To check the list of node in swarm you should execute next command:
```
docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
f7ysphew4er8dkmvefkbot4vc *   swarm-manager       Ready               Active              Leader              18.06.1-ce
```
Let's join our free worker to Docker Swarm Cluster as well. First we need to login to our worker VirtualBox machine by ssh
```
vagrant ssh swarm_worker
```
Then execute command which returned our manager during initialization:
```
docker swarm join --token SWMTKN-1-59uvhwxzz34cj5t1kiykwidrzg4afepdz0r96rjk5e1bcxx9tz-dmfs8pqr71ndranwm5xy68of3 192.168.56.101:2377
```

Let's check the list of nodes now. You need to execute this command in swarm manager terminal:
```
In swarm_manager terminal:
docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
f7ysphew4er8dkmvefkbot4vc *   swarm-manager       Ready               Active              Leader              18.06.1-ce
0anl5bgtyxjtui3xwszab89j9     swarm-worker        Ready               Active                                  18.06.1-ce
```
You can see that now our cluster contains 2 machines(worker and manager).
To deploy our simple application described in docker-stack.yml file we need to execute command below in docker swarm terminal:
```
In swarm_manager terminal:
docker stack deploy -c=/vagrant_data/docker-stack.yml swarm-test
```
Let's check the list of stacks in our cluster.
```
docker stack ls
NAME                SERVICES            ORCHESTRATOR
swarm-test          2                   Swarm
```
To see more detailed information about services you can use command below:
```
docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                 PORTS
wst59z76lezz        swarm-test_nginx    replicated          0/5                 ningx:1.15.8-alpine   
vwifnjnrpf55        swarm-test_redis    replicated          1/1                 redis:5.0.3      
```
You can see that nginx has 0 replicas(your number maybe different). This mean that deploy is still in progress.
If we will execute this command one more time, then we will see that amount of containers 
increased to 4 for nginx service.
```
docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                 PORTS
wst59z76lezz        swarm-test_nginx    replicated          4/5                 nginx:1.15.8-alpine   *:80->80/tcp
vwifnjnrpf55        swarm-test_redis    replicated          1/1                 redis:5.0.3      
```
And finally we have the desired state of replicas, it's 5 now.
```
docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                 PORTS
wst59z76lezz        swarm-test_nginx    replicated          5/5                 nginx:1.15.8-alpine   *:80->80/tcp
vwifnjnrpf55        swarm-test_redis    replicated          1/1                 redis:5.0.3 
```
As you remember in docker-stack file we asked docker to run redis container on manager role, but all 5 containers for nginx service will be deployed on manager and worker machines based on predefined docker rules.
To check which containers now running on manager machine you can use:
```
docker node ps swarm-manager
ID                  NAME                 IMAGE                 NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
m1wczelcwyaz        swarm-test_redis.1   redis:5.0.3           swarm-manager       Running             Running 6 seconds ago                       
kqw6ol3olh3j        swarm-test_nginx.3   nginx:1.15.8-alpine   swarm-manager       Running             Running 8 seconds ago                       
qgvd65c553qt        swarm-test_nginx.5   nginx:1.15.8-alpine   swarm-manager       Running             Running 8 seconds ago 
```
For worker machine:
```
docker node ps swarm-worker
ID                  NAME                 IMAGE                 NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
1qpe5vaz5fha        swarm-test_nginx.1   nginx:1.15.8-alpine   swarm-worker        Running             Running 2 minutes ago                       
haev50eeha20        swarm-test_nginx.2   nginx:1.15.8-alpine   swarm-worker        Running             Running 2 minutes ago                       
z45nfu05ip3w        swarm-test_nginx.4   nginx:1.15.8-alpine   swarm-worker        Running             Running 2 minutes ago 
```
We can always tell Docker to run nginx containers only on machines with worker role or define any other rules.
What will be if worker machine will be destroyed for any reason? In this case manager will recognize this and will run missed containers on manager machine.
Let's do it and stop worker machine:
```
vagrant halt swarm_worker
```
Swarm manager will recognize it soon and will make swarm-worker host as Down:
```
docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
f7ysphew4er8dkmvefkbot4vc *   swarm-manager       Ready               Active              Leader              18.06.1-ce
0anl5bgtyxjtui3xwszab89j9     swarm-worker        Down                Active                                  18.06.1-ce
```
Let's check the state of containers on worker machine
```
docker node ps swarm-worker
ID                  NAME                 IMAGE                 NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
1qpe5vaz5fha        swarm-test_nginx.1   nginx:1.15.8-alpine   swarm-worker        Shutdown            Running 54 seconds ago                       
haev50eeha20        swarm-test_nginx.2   nginx:1.15.8-alpine   swarm-worker        Shutdown            Running 54 seconds ago                       
z45nfu05ip3w        swarm-test_nginx.4   nginx:1.15.8-alpine   swarm-worker        Shutdown            Running 6 minutes ago
```
As you can see all has "Shutdown" state.
Let's check running containers on manager machine
```
docker node ps swarm-manager
ID                  NAME                 IMAGE                 NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
ybe5v33b5nfh        swarm-test_nginx.1   nginx:1.15.8-alpine   swarm-manager       Running             Running 15 minutes ago                       
m1wczelcwyaz        swarm-test_redis.1   redis:5.0.3           swarm-manager       Running             Running 22 minutes ago                       
rb67ox7i83iw        swarm-test_nginx.2   nginx:1.15.8-alpine   swarm-manager       Running             Running 15 minutes ago                       
kqw6ol3olh3j        swarm-test_nginx.3   nginx:1.15.8-alpine   swarm-manager       Running             Running 22 minutes ago                       
rgxpc9ooc8wn        swarm-test_nginx.4   nginx:1.15.8-alpine   swarm-manager       Running             Running 15 minutes ago                       
qgvd65c553qt        swarm-test_nginx.5   nginx:1.15.8-alpine   swarm-manager       Running             Running 22 minutes ago 
```
As you can see manager ran missed containers from worker machine.
What will be if worker machine return back online? In this case manager also recognize it and deploy some amount of nginx containers back to worker machine when new deploy will be done or desired state of containers will be changed
```
vagrant up swarm_worker

docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
f7ysphew4er8dkmvefkbot4vc *   swarm-manager       Ready               Active              Leader              18.06.1-ce
0anl5bgtyxjtui3xwszab89j9     swarm-worker        Ready               Active                                  18.06.1-ce

docker node ps swarm-manager
ID                  NAME                 IMAGE                 NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
ybe5v33b5nfh        swarm-test_nginx.1   nginx:1.15.8-alpine   swarm-manager       Running             Running 20 minutes ago                       
m1wczelcwyaz        swarm-test_redis.1   redis:5.0.3           swarm-manager       Running             Running 27 minutes ago                       
rb67ox7i83iw        swarm-test_nginx.2   nginx:1.15.8-alpine   swarm-manager       Running             Running 20 minutes ago                       
kqw6ol3olh3j        swarm-test_nginx.3   nginx:1.15.8-alpine   swarm-manager       Running             Running 27 minutes ago                       
rgxpc9ooc8wn        swarm-test_nginx.4   nginx:1.15.8-alpine   swarm-manager       Running             Running 20 minutes ago                       
qgvd65c553qt        swarm-test_nginx.5   nginx:1.15.8-alpine   swarm-manager       Running             Running 27 minutes ago
```

Let's initiate this desired state with scale command to change the amount of replicas for nginx service:
```
docker service scale swarm-test_nginx=7

docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                 PORTS
wst59z76lezz        swarm-test_nginx    replicated          7/7                 nginx:1.15.8-alpine   *:80->80/tcp
vwifnjnrpf55        swarm-test_redis    replicated          1/1                 redis:5.0.3 

docker node ps swarm-worker
ID                  NAME                 IMAGE                 NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
1qpe5vaz5fha        swarm-test_nginx.1   nginx:1.15.8-alpine   swarm-worker        Shutdown            Shutdown 3 minutes ago                       
haev50eeha20        swarm-test_nginx.2   nginx:1.15.8-alpine   swarm-worker        Shutdown            Shutdown 3 minutes ago                       
z45nfu05ip3w        swarm-test_nginx.4   nginx:1.15.8-alpine   swarm-worker        Shutdown            Shutdown 3 minutes ago                       
vhnskojmbdaw        swarm-test_nginx.6   nginx:1.15.8-alpine   swarm-worker        Running             Running 20 seconds ago                       
a42a3gn8a4df        swarm-test_nginx.7   nginx:1.15.8-alpine   swarm-worker        Running             Running 20 seconds ago
```
You can see that 2 new nginx containers were deployed on worker machine.

Let's check our app access, just open browser and try to access 80 port for manager or worker IP address.
192.168.56.101 - manager
192.168.56.102 - worker
Both urls should return default Nginx page

To remove our stack you can use command below:
```
docker stack rm swarm-test
```
If we want worker machine to leave the swarm, then in worker terminal we need to execute:
```
docker swarm leave

# output
Node left the swarm.
```
If you want manager to leave the cluster then you should pass --force param
```
docker swarm leave
Error response from daemon: You are attempting to leave the swarm on a node that is participating as a manager. Removing the last manager erases all current state of the swarm. Use `--force` to ignore this message.

docker swarm leave --force
Node left the swarm
```
To stop 2 VirtualBox machine you should use command below:
```
vagrant halt
```
To completely remove 2 VirtualBox machines you should execute:
```
vagrant destroy
    swarm_worker: Are you sure you want to destroy the 'swarm_worker' VM? [y/N] y
==> swarm_worker: Destroying VM and associated drives...
    swarm_manager: Are you sure you want to destroy the 'swarm_manager' VM? [y/N] y
==> swarm_manager: Destroying VM and associated drives...
```
