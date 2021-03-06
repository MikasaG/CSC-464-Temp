# Consensus in Redis Cluster
We use Redis not only for cache, but also storing important data, and we build up a Master/Slave replication topology to guarantee data security.

Master/Slave architecture works well, but sometimes we need a more powerful high availability solution. If master is down, we must check this immediately, reselect a new master from the slaves and do failover.

The official Redis supplies a solution named redis-sentinel, which is where the **Raft** protocol is used.
## My Project
My project is about implementing a **seckill shopping system** . 
On Black Friday, lots of online shop owners will set a extremely low price and low stocks to several goods, there will be a lot of customers competing for these items, the purpose of my system is to correctly handle purchase request and reject requests when a commodity is sold out.

When designing this system, Considering the demand of highly concurrent access to the database over different places, I chose to use **Distributed Redis Cluster** as the cache layer. The  _high availability_ provided by the master and slave model of Redis can also give customers perfect user experience.
## Consensus  in Redis Cluster - Sentinel (Raft Application)
### Master and Slave Model (AP solution)
According to the CAP theorem,  any net­worked shared-data system cannot have all of the following desirable properties:  Consistency,  Availability  and Partition Tolerance. The Master and Slave Model used by Redis Cluster here can be categorized as an AP system.  The slave regularly backs up its master's data and when a master node fails, the cluster will promote one of its slave as the new master.
### Sentinel Structure
the work of a single sentinel node is not complex: checking master every second, and do failover when master is down. But, when the sentinel is down too, how do we do?we may need to using sentinel cluster to handle this situation, if one sentinel is down, other sentinel will still work. But if two sentinels in the cluster both see the master is down, and do failover at same time, sentinel1 may select slave1 as master, but sentinel2 may select slave2 as master, this may be a horrible thing for us.

A common use way is to elect a leader sentinel in the cluster and let it monitor and do failover. This requires a consensus on our system. So we need a consensus algorithm (**Raft** is used here) helping us do this thing.
### Paxos and Raft

Paxos may be the most famous consensus algorithm in the world, many companies use it in their distributed system. However, Paxos is very hard to understand and if you write a paxos lib by yourself, you even cann’t testify its correctness easily. 

Raft was born on 2013 in Stanford, it’s very new but awesome. Raft is easy to understand, everyone reading the Raft paper can write its own Raft implentation easily than Paxos. Now many projects use Raft, like Etcd, InfluxDB, CockroachDB, etc…

## My Implementation 
I created a **Redis Cluster** with 9 working nodes and 1 sentinel node. I used  network ports in Windows to build this cluster, avoiding the resource consumption of virtual machines or physical servers. So I have 9 ports in use, three of them are masters, each master have 2 slaves (followers/replicas).
|Master | Slave
|-------- | -----
|7000 | 7003,7004
|7001 | 7005,7006
|7002 | 7007,7008
Run redis on a single port:
![Run redis on a single port](https://raw.githubusercontent.com/MikasaG/CSC-464/master/Assignment%202/Consensus%20in%20Redis%20Cluster/images/run%20redis%20on%20single.png)

Make all nodes together, it becomes a distributed system now **(Redis Cluster)**.
![Creating Cluster](https://raw.githubusercontent.com/MikasaG/CSC-464/master/Assignment%202/Consensus%20in%20Redis%20Cluster/images/creating%20cluster.png)
As you can see from the picture above, M and S indicates master and slave respectively.

Then I create a **sentinel node** to listen on port 7000, and run it manually.
![Creating sentinel node](https://raw.githubusercontent.com/MikasaG/CSC-464/master/Assignment%202/Consensus%20in%20Redis%20Cluster/images/run%20sentinel.png)
At this point, our cluster setup is complete.

## Testing Method and Results
Firstly, We should test **get(key)** and **set(key, value)** operations when no master nodes fails.
![before 7000 down](https://raw.githubusercontent.com/MikasaG/CSC-464/master/Assignment%202/Consensus%20in%20Redis%20Cluster/images/before%207000%20down.png)
Everything are good so far.

Then I manually turn down the port 700, which means our first master nodes is offline now.
check the log of sentinel node:
![log of sentinel node](https://raw.githubusercontent.com/MikasaG/CSC-464/master/Assignment%202/Consensus%20in%20Redis%20Cluster/images/sentinel%20elect%20and%20switch%20master.png)
From the log we can know that, the sentinel detect the master node **fail** at *16:35:30*, then it **vote** itself as a new sentinel leader at *16:35:36* and **switch** master to node 7004 at *16:35:38*. We can conclude that the cluster successfully elect out a new sentinel leader and complete switching work when a primary node fail.

Now, I test whether a client can feel the failure of a primary node. Assume an user put a key-value pair ("aa" - 1)into node 7000 before it fails.
![before 7000 down](https://raw.githubusercontent.com/MikasaG/CSC-464/master/Assignment%202/Consensus%20in%20Redis%20Cluster/images/before%207000%20down.png)

After node 7000 fails:
![after 7000 down](https://raw.githubusercontent.com/MikasaG/CSC-464/master/Assignment%202/Consensus%20in%20Redis%20Cluster/images/after%207000%20down.png)
The distributed cluster automatically redirect user's **get** operation to node 7004, which is the new master node now. User activity is completely unaffected.