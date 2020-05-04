# NanoBoost

In the real world, most computation jobs can be treated as DAG, for example, video compression can be expedited by first breaking a large piece of video into multiple small chunks, compressing individually and merging them afterwards. We propose a framework NanoBoost that can easily break a single machine job and distribute it across LAN on commodity devices, speeding up the execution and scalability of the job. 

NanoBoost is a scalable, distributed fault-tolerant DAG execution framework with simple user interfaces. It supports dynamic membership change so nodes are plugable -- easy to join and leave the workpool. We keep raspberry pi in mind so NanoBoost will support normal Linux distros and also Raspbian, tasks will be distributed to workers based on task size and worker’s performance.

# Architecture
In this section we'll describe the system architecture. The word "task" are carefully chosed to reduce confusion.

## Players in the system
Nanoboost contains 3 types of players, all players are connected under the same LAN.
1. **Master**

Master has 3 primary responsibilities:
* Assign *task* to agents for execution.
* Maintain a *flow graph*  that contains all *tasks* to execute. 
* Maintain an executin *frontier* (execution progress).

In Nanoboost, we assume master can failover, and agents will be elected to become new master.

2. **Agent**

Agent is the simplest player, its sole job is to execute *task* it gets assigned. However, it can also become master when master fails.

3. **NAS**
Network Attached Storage serves as the central storage location, where we'll store intermediate data, input/output. Choosing a NAS simplified the design by not managing file transmittion between master and agents. In Nanoboost, we assume that NAS doesn't fail, this can be achieved by multiple backup.
![players](https://docs.google.com/drawings/d/e/2PACX-1vSFXql0Cossfx5Ca83LCHtUfP4QiFgLaEr20eytbUiZ54MGCAAHwpcLVXvcPj0mDkNZfriMN13eXoes/pub?w=960&h=720)

## Software Architecture
This subsection describes the software architecture of NanoBoost, where we divide the software stack as below.
![Architecture](https://docs.google.com/drawings/d/e/2PACX-1vRurl49ozZYTcWlaQMzBd2kBN8qbBRsgD5-r5edCAZ2DTafS9CTfMUn4Z6z4CgT42lrDFVy1Rpdw1C_/pub?w=960&h=720)

NanoBoost can be devided into 3 layers, from bottom up, they are:
1. Consensus Layer

The natural question is, why do we need this layer? This is the core layer to answer 3 questions:
* Am I the master to schedule the *task* execution?
* Who are the members of the system?
* What's the progress of execution.

In this layer we implemented consensus functionalities that does **master election**, **frontier storage** and **membership management**. This layer provides 3 APIs towards upper layer, which are:
