#####  

## 3. Installation and Deployment

### 3.1 Software and hardware environment requirements

ZNBase is a distributed database product created by Inspur, with features such as strong consistency, highly available distributed architecture, distributed horizontal expansion, high performance, and enterprise-level security. It can be deployed and run in X86 architecture server environment, ARM architecture server environment and mainstream virtualization environment, and supports most mainstream hardware networks.

#### 3.1.1 Linux OS

Yunxi ZNBase database supports mainstream operating systems: Linux, Kylin, Galaxy Kylin, UOS, etc.

Linux OS：

| Linux  OS platform       | Version                 |
| ------------------------ | ----------------------- |
| CentOS                   | Version 7.3 and above   |
| Kylin                    | Version 10 and above    |
| Ubuntu LTS               | Version 18.04 and above |
| Red Hat Enterprise Linux | Version 7.3 and above   |

#### 3.1.2  Hardware Requirements

Yunxi ZNBase database supports mainstream hardware systems: X86, ARM, Haiguang, PHYTIUM, Zhaoxin, etc.

The minimum specifications of the cluster that the Yunxi ZNBase database can run are as follows:

| Deployment medium                   | Minimum specification        | Remarks                                                      |
| ----------------------------------- | ---------------------------- | ------------------------------------------------------------ |
| Physical machine or virtual machine | Three nodes and three copies | Each node is deployed on a separate physical machine or virtual machine |

Minimum server resource requirements:

| CPU                                                    | Hard disk                                                    | File system                       |
| ------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------- |
| A single node should have at least 2 CPUs and 8 GB RAM | SSD disk is required, NVME SSD is recommended, data disk does not need to be Raid, single disk is above 480 GB | Recommend to use EXT4 file system |

<!--Note: According to different scenarios such as data volume and complexity, configure more hardware resources.-->

**Containerized Deployment**：Yunxi ZNBase database supports container deployment. The Yunxi ZNBase database itself is packaged with DOCKER, relying on the Inspur cloud native platform and ecology, the database supports Kubernetes cloud native interface, which can realize one-click deployment and container arrangement, rapid construction, management and monitoring.

#### 3.1.3   Software Requirements

- The operating system of each node in the cluster needs to install the necessary software packages, and the database files need to rely on GLIBC, LIBNCURSES, TZDATA.
- The operating system of each node in the cluster needs to install an NTP software package or other clock synchronization software to ensure that the operating system time of each node is consistent.
- Choose to install the HAPROXY load balancing function according to business needs, and the version is not less than 1.5.0.

#### 3.1.4 Web Browser Configuration Requirements

  ZNBase provides the AdminUI database console, which visually displays various indicators of the database cluster, and supports the newer version of Google Chrome to access the monitoring portal.

### 3.2 Deployment Environment Check

**This part introduces the checking and setting of operating system consistency, cluster node time consistency, database port and file handle number during ZNBase database cluster deployment.**

#### 3.2.1 Operating System Version Consistency:

Check the consistency of the operating system version in each node of the cluster. The command to view the Linux system version is as follows:

#lsb_release –a 

> No LSB modules are available. 
> Distributor ID: Ubuntu 
> Description: Ubuntu 16.04.6 LTS 
> Release: 16.04 
> Codename: xenial 

#### 3.2.2 The system time of the nodes in the cluster is consistent

A medium-strength clock synchronization mechanism is required in cluster nodes to maintain data consistency. When a node detects that the error between its own machine time and the machine time of at least 50% of the nodes in the cluster exceeds 80% of the maximum allowable time error of the cluster (default 500ms), the node will automatically stop. This can avoid violating data consistency and leading to the risk of reading and writing old data. Therefore, NTP or other clock synchronization software must be run on each node to prevent the clock from drifting too far.

#### 3.2.3 Whether the default port of the database service is occupied

#lsof -i:26257 
> COMMAND     PID  USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME 
> drdb 10583 jesse   12u  IPv6 3263391      0t0  TCP *:26257 (LISTEN)

If the port number conflicts with the database default port number, you can use the kill -9 pid_value command under the root user to terminate the conflicting process or modify the default port number when installing the database.

#### 3.2.4 Modify Linux system file handle limit

The database uses a large number of file handles, which usually exceeds the number of file handles available in LINUX by default. Therefore, it is recommended to modify the number of file handles for each database node: the maximum number of file handles needs to be set to at least 1956 (each store requires 1700 file handles, 256 for the network), nodes below this threshold will not be able to start database services. It is recommended to configure the maximum number of file handles as UNLIMITED, or set the value to 15000 (where each store requires 10000 file handles and 5000 for the network) or higher to support the performance requirements of database cluster growth. If there are 3 stores on a node, we recommend changing the hard limit to at least 35000 (10000 for each store and 5000 for the network).

Taking Cent OS 7 as an example, the method to modify the number of handles is:

- Edit "/etc/security/limits.conf" and append the following content after the file:

        * soft nofile 35000
        * hard nofile 35000

- Save and close the file.

- Restart the system for the modified content to take effect.

- Execute "ulimit -a" command to confirm the modification result

- Modify the system-wide limit: You need to ensure that the system-wide maximum handle limit is at least 10 times the above single process limit. View the number of file handles in the system: "cat /proc/sys/fs/file-max".


Increase the system-wide limit value as needed: "echo 350000> /proc/sys/fs/file-max" .

<!--Note: Regarding the operating system maximum file handle value, the database only takes the hard limit value, so there is no need to adjust the soft limit value.-->

### **3.3**  Configuration Topology

This part describes the minimal deployment topology of the ZNBase cluster.

The ZNBase architecture diagram is shown below：

![image-20210104150918258](NEWSQL开源手册大纲以及输出标准(沈鹏部分)-English edition .assets/image-20210104150918258.png)

Topology information: the smallest specification of the cluster, three nodes and three copies

| Node  | Node Configuration    | IP （10.1.1.X as example） | Configuration                                 |
| ----- | --------------------- | -------------------------- | --------------------------------------------- |
| Node1 | 32 CPUs and 64 GB RAM | 10.1.1.1                   | Default port and global catalog configuration |
| Node2 | 32 CPUs and 64 GB RAM | 10.1.1.2                   | Default port and global catalog configuration |
| Node3 | 32 CPUs and 64 GB RAM | 10.1.1.3                   | Default port and global catalog configuration |

### **3.4**  Installation start

**This part introduces database deployment and startup, mainly divided into two ways for users to choose.**

● Deploy and start ZNBase cluster in security mode

● Deploy and start ZNBase cluster in non-security mode

#### 3.4.1 Precondition

- Ensure that the system time of each node in the cluster is synchronized.

- Ensure that each node in the cluster can access each other through SSH.

- Network configuration allows TCP communication on ports 26257 and 8080.


#### 3.4.2 Security mode deployment and startup

##### **1.**     **Getting the database executable file**

a)     Obtain the ZNBase database file and upload it to "/usr/local/bin" under the PATH path

b)    Generate local certificate: Create two new directories: "/opt/certs" is used to store the generated CA certificate and all nodes and client certificate and key files, some of which will be transferred to the node machine. "/opt/my-safe-directory" is used to store the generated CA key file, which will be used when creating certificates and keys for nodes and users later. 

`$ mkdir /opt/certs`

`$ mkdir /opt/my-safe-directory`

**2.**     Generate Local Certificate 

c)     Create CA certificate and key:

`$ drdb cert create-ca --certs-dir=/opt/certs --ca-key=/opt/my-safe-directory/ca.key`

d)    Create a certificate and key for the first node:

`$ drdb cert create-node <node1 internal IP address> <node1 external IP address> <node1 hostname>  <other common names for node1> localhost 127.0.0.1 <load balancer IP address> <load balancer hostname>  <other common names for load balancer instances> --certs-dir=/opt/certs --ca-key=/opt/my-safe-directory/ca.key `

e)     Transfer the CA certificate, node certificate and key to the first node:

`$ ssh <username>@<node1 address> "mkdir /root/certs" `

`$ scp /opt/certs/ca.crt /opt/certs/node.crt /opt/certs/node.key <username>@<node1 address>:/root/certs `

f)     Delete the local node certificate and key:

`$ rm /opt/certs/node.crt /opt/certs/node.key`

g)     Create a certificate and key for the second node:

`$ drdb cert create-node <node2 internal IP address> <node2 external IP address> <node2 hostname>  <other common names for node2> localhost 127.0.0.1 <load balancer IP address> <load balancer hostname>  <other common names for load balancer instances> --certs-dir=/opt/certs --ca-key=/opt/my-safe-directory/ca.key `

h)    Transfer the CA certificate, node certificate and key to the second node:

`$ ssh <username>@<node2 address> "mkdir /root/certs" `

`$ scp /opt/certs/ca.crt /opt/certs/node.crt /opt/certs/node.key <username>@<node2 address>:/root/certs`

i)      For each other cluster node that needs to install a certificate, please repeat steps f)-h).

j)     Create a client certificate and key for the root user:

`$ drdb cert create-client root --certs-dir=/opt/certs --ca-key=/opt/my-safedirectory/ca.key `

k)    Transfer the certificate and key to the machine where you want to execute the database command. The machine can be a node in the cluster or outside the cluster. The machine with the certificate can use the root account to execute the database command, which can be accessed through the node machine Cluster:

`$ ssh <username>@<workload address> "mkdir /root/certs" $ scp /opt/certs/ca.crt /opt/certs/client.root.crt /opt/certs/client.root.key <username>@<workload address>:/root/certs` 

l)     If you want to run database client commands on some other machine in the future, you need to copy the root user's certificate and key to that node. Only nodes with root user certificates and keys can access the cluster.

**3.**     **Database Startup**

m)   SSH to the node machine that needs to start the service, obtain the database executable file, and upload it to the specified directory.

`$ cp -i drdb-v****.linux-amd64/drdb /usr/local/bin/ `

n)    Execute the database start command.

`$ drdb start --certs-dir=/root/certs --store=/opt/node1 --advertise-addr=<node1 address>:26257 --listen-addr=<node1 address>:26257 --http-addr=<node1 address>:8080 --join=<node1 address>,<node2 address>,<node3 address> -cache=.25 --max-sql-memory=.25 --background `

o)    Perform m)-n) steps for each node that needs to be added to the cluster.

<!--Note: The parameter description of the database opening command is as follows:-->

| **Parameter**              | **Description**                                              |
| -------------------------- | ------------------------------------------------------------ |
| --certs-dir                | Specify the directory to store "ca.crt", "node.crt", "node.key" files. |
| --store                    | Specify the name of the directory where the database files are stored. |
| --advertise-addr           | Specify the IP address or hostname for other nodes in the cluster to access the node. |
| --listen-addr              | Specify the listening IP and port of the database service.   |
| --http-addr                | Specify the IP and port of the cluster monitoring tool.      |
| --join                     | Specify the addresses and ports of the cluster 3-5 initialization nodes. |
| --cache   --max-sql-memory | Increase the size of node cache and temporary SQL memory to 25% of operating system memory to optimize read performance and in-memory SQL execution. |
| --background               | Specify background operation.                                |

**4.**     **Initialize the Cluster**

p)    Execute the database init command on the first node of the cluster.（The node needs to have the certificate and key of the root user）

`$ drdb init --certs-dir=/root/certs --host=<address of any node> `

#### 3.4.3  Non-Security mode deployment and startup

**1.**     **Getting the database executable file**

a)     Obtain the ZNBase database file and upload it to "/usr/local/bin" under the PATH path.

**2.**     **Database Startup**

b)    SSH to the node machine that needs to start the service, obtain the database executable file, and upload it to the specified directory.

c)     Execute the database start command:

`$ drdb start --insecure --store=/opt/node1 --advertise-addr=<node1 address>:26257 --listen-addr=<node1 address>:26257 --http-addr=<node1 address>:8080 --join=<node1 address>,<node2 address>,<node3 address> -cache=.25 --max-sql-memory=.25 --background `

d)    Repeat steps b)-c) on other nodes in the cluster.

<!--Note: The parameter description of the database opening command is as follows:-->

| **Parameter**              | **Description**                                              |
| -------------------------- | ------------------------------------------------------------ |
| --insecure                 | Specify to start the cluster in non-security mode            |
| --store                    | Specify the name of the directory where the database files are stored |
| --advertise-addr           | Specify the IP address or hostname for other nodes in the cluster to access the node |
| --listen-addr              | Specify the listening IP and port of the database service    |
| --http-addr                | Specify the IP and port of the cluster monitoring tool       |
| --join                     | Specify 3-5 addresses and ports of the cluster initialization nodes |
| --cache   --max-sql-memory | Increase the size of node cache and temporary SQL memory to 25% of operating system memory to optimize read performance and in-memory SQL execution |
| --background               | Specify background operation                                 |

**3.**     **Initialize the cluster**

e)    Execute the database ”init“ command on the first node of the cluster

` $ drdb init --insecure --host=<address of any node> `

<!--**Note: Risks of non-secure mode clusters:**-->

- The cluster is open to any client and can access the IP address of any node in the cluster.

- Any user, even the root user, can access the cluster without a password.

- Any user can access the cluster as the root user and can read and write any data in the cluster.

- There is no network encryption or authentication, so it lacks confidentiality.


#### 3.4.4 Check after deployment

**1.**     Create NODETEST database

   a)     Open interactive shell

​    #security-mode startup

​ `   $drdb sql --cert-dir=/root/certs --host=<address of any node>`

​    #non-security-mode startup

​   ` $drdb sql --insecure --host=<address of any node>`

   b)    创建NODETEST数据库

> create database nodetest;

   c)     使用\q或CTRL+D退出

**2.**     Connect to another node to check the NODETEST database

   a)     Open interactive shell

​    #security-mode startup

​  `  $drdb sql --cert-dir=/root/certs --host=<address of any node>`

​    #non-security-mode startup

​  `  $drdb sql --insecure --host=<address of any node>`

   b)    Check the database

> SHOW DATABASES;

   c)     Use `\q` or `CTRL+D` to exit

#### 3.4.5 Configure HAProxy Load Balancing

Each node in a database cluster is a SQL gateway for the cluster. However, in order to ensure the performance and reliability of the client, it is recommended to use the load balancing function, and the ZNBase database has a built-in command to generate the HAPROXY configuration file that can be used with the running cluster:

- Performance: The load balancer distributes client traffic across nodes. This prevents one node from handling excessive traffic and improves overall cluster performance (queries per second).

- Reliability: The load balancer can try to avoid the impact of node failure in the cluster on the client. If a node fails, the load balancer will redirect client traffic to an available node.

- The configuration method is as follows:


  a)     Generate HAPROXY configuration file

   `$ drdb gen haproxy --certs-dir=certs --host=<address of any node>` 

  By default, the HAPROXY.CFG file is automatically generated, and the configuration file is as follows:

> Global  
>            maxconn 4096 
>            defaults  mode tcp 
>            Timeout values should be configured for your specific use. 
>            See: https://cbonte.github.io/haproxy-dconv/1.8/configuration.html#4timeout%20connect 
>            timeout connect     10s
>            timeout client      1m
>            timeout server      1m 
>            TCP keep-alive on client side. Server already enables them. option              
>            clitcpka listen psql bind :26257 
>            mode tcp 
>            balance roundrobin 
>            option httpchk GET /health?ready=1 
>            server drdb1 <node1 address>:26257 check port 8080 
>            server drdb2 <node2 address>:26257 check port 8080 
>            server drdb3 <node3 address>:26257 check port 8080

  b)    Upload the CFG file to the HAPROXY machine to be run

  `$ scp haproxy.cfg <username>@<haproxy address>:~/ `

  c)     SSH to the running machine and install

  `$ apt-get install haproxy `

 d)    Start HAPROXY, use -f to point to the CFG file

  `$ haproxy -f haproxy.cfg `

 e)     To use multiple HAPROXY instances, please repeat steps a)-d)

### 3.5  Performance Test 

#### 3.5.1 TPC-C test on ZNBase database

This part introduces how to perform TPC-C test on ZNBase database. It mainly tests the SQL statement support in the standard model TPCC. The SQL statement structure in the TPCC model is complex. By verifying the SQL operation support in the standard model, the database pair can be judged. SQL support ability.

##### **1.**   **Database standard model TPCC support status**

###### 1.1 Prepare the environment

a)   One set of three-node cluster environment in safe mode and non-safe mode.

b)   The database has been started and is in normal working condition.

c)   The built-in tool workload loads TPCC test data

d)   Built-in tool workload for TPCC performance test

###### **1.2  Test command**

- Load data (security mode):

  `drdb workload init tpcc 'postgresql://root@localhost:26257?sslcert=certs/client.root.crt&sslkey=certs/client.root.key&sslmode=verify-full&sslrootcert=certs/ca.crt'`

- Perform the test (security mode):

  `drdb workload run tpcc --duration=1m 'postgresql://root@localhost:26257?sslcert=certs/client.root.crt&sslkey=certs/client.root.key&sslmode=verify-full&sslrootcert=certs/ca.crt'`

- Load data (non-security mode):

  `drdb workload init tpcc 'postgresql://root@localhost:26257?sslmode=disable'`

- Perform the test (non-security mode):

  `drdb workload run tpcc --duration=1m 'postgresql://root@localhost:26257?sslmode=disable`

######  **1.3  Result analysis**

   Run TPCC performance test through tool, support TPCC model SQL

   ![image-20201216203749858](NEWSQL开源手册大纲以及输出标准(沈鹏部分).assets/image-20201216203749858.png)

##### **2.**   Performance of database standard model TPCC

###### **2.1 Prepare the environment**

- One set of cluster environment in safe mode and non-safe mode.

- Build a large-scale cluster of 200 nodes.

- Load TPCC test data of 4000 bins.

- The database has been started and is in normal working condition.

- The built-in tool workload loads TPCC test data.

- Built-in tool workload for TPCC performance test.

###### **2.2 Test command**

-  Load data (security mode):

  `drdb workload init tpcc 'postgresql://root@localhost:26257?sslcert=certs/client.root.crt&sslkey=certs/client.root.key&sslmode=verify-full&sslrootcert=certs/ca.crt'`

- Perform the test (security mode):

  `drdb workload run tpcc --duration=1m 'postgresql://root@localhost:26257?sslcert=certs/client.root.crt&sslkey=certs/client.root.key&sslmode=verify-full&sslrootcert=certs/ca.crt'`

- Load data (non-security mode):

  `drdb workload init tpcc 'postgresql://root@localhost:26257?sslmode=disable'`

- Perform the test (non-security mode):

  `drdb workload run tpcc --duration=1m 'postgresql://root@localhost:26257?sslmode=disable`

###### **2.3  Result analysis**

​        200 database nodes, 4000 warehouse TPCC data, tmpC about 44662
