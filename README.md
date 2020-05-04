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
1. ***Consensus Layer***

The natural question is, why do we need this layer? This is the core layer to answer 3 questions:
* Am I the master to schedule the *task* execution?
* Who are the members of the system?
* What's the progress of execution.

In this layer we implemented consensus functionalities that does **master election**, **frontier storage** and **membership management**. This layer provides 2 APIs towards upper layer, which are:
* Get/SetFrontier: Master will get frontier when it initialize, or set frontier once frontier is updated.
* GetMembers: This will return all members in cluster.

Besides that, it also manages mastership information, and notify upper layer (though NanoBoost's MastershipChange RPC) whenever a mastership change happens.

2. ***NanoBoost Layer***

This layer consists of **master** and **agents**.
### Master
We design master as a pipeline, which makes it easier to implement. A master contains 4 modules:
* **Listener**: This module exposes 2 RPCs - *TaskDone* and *MastershipChange*. When TaskDone is invoked, it will forward this information to scheduler and expose more frontier.
* **Scheduler**: This module trackes what's the remaining tasks and what's the frontier. It maintains in-memory NanoGraph and frontier, and schedule the execution order. Whenever a new frontier is exposed, it will update the frontier in consensus layer by invoking SetFrontier. Note that, we store the entire frontier, thus the frontier we are updating are idempotent so that the frontier stored in Consensus will never gets dirty.
* **TaskQueue**: This module stores the frontier, it can be implemented as either a queue or a priority queue. It's in-memory and will be consumed by Dispatcher.
* **Dispatcher**: This module dispatches job to agents in a load-balancing manner. It periodically gets membership information from Consensus layer, and invokes agents' Execute RPC. Note that we don't maintain a session with other agents, instead agents reply to master through TaskDone API in listener.

### Agent
Agent should be simple. In NanoBoost agent contains one RPC, which is Execute. It Execute master's scheduled task and reply through TaskDone.

3. ***User Space***

We intended to enable NanoBoost to execute any DAG graph. User space API mainly answers 2 questions:
* What does the execution graph (NanoGraph) look like?
* What does each vertex do.
Thus we have 2 following components:
### Nano
Nano is the vertex in the graph, it basically takes some inputs in NAS, executed based on how user defines, and output the executed result(s) back to NAS. 

### NanoGraph
NanoGraph defines the execution graph. It links Nanos together and defined the input and output of it. For ease of implementation we design it as a static graph, in stage 2 we'll extend it to be dynamically generated. 

Note that, a user has to provide both of the above implementation in order to NanoBoost to run.
