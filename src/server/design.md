**Q:** Are client-keeps of objects managed by Scheduler or just the Reactor?

## Reactor

A common loop turn elements:

### Fetch worker updates

Receive: states of tasks, objects and workers, updating graph states. Moreover:

* Finished tasks vacate worker resources. New tasks are not assigned at this point.

* Running tasks are just updated.

* Finished objects are removed from their consumer tasks requirements. These tasks may 
become ready and possibly placed into scheduled_ready on their worker. 

* Located tasks (copies) are just updated.

* Worker errors are not handled in any way, just panic.

All updates are piled up for the scheduler.

### Fetch client updates

Receive: Submits, unkeeps, waits, fetches.

Submits are added to graph, added to sceduler events, verified for consistency. 

Unkeeps are applied directly to graph. If a Finished data object has no schedules, 
unassign it and remove it.

**TODO:** specify waits and fetches of unfinished data.

### Run scheduler

Scheduler is ran only with certain periodicity and depending on pending updates (new 
submit?) and last scheduler run-time (do not spend too much time in scheduling).  

**Idea:** Run scheduler only if the last run was more than *threshold* ago. The threshold
values are given for new submits present (e.g. 10ms, but not zero), for updates present
(e.g. 100ms) and no updates (e.g. 1s). The threshold may be increased in case the last
scheduling took very long (in order to spend e.g. <50% time scheduling).  

**TODO:** Specify timing conditions.

### Assign tasks and objects to workers

For every worker, assign or un-assign any objects directly.

For every worker, run through scheduled ready workers and overbook the worker by 
assigning the tasks and objects. (Sending info on all objects not assigned to worker.)

## Scheduler interaction

The scheduler has ful read access to the graph. The plan is reflected in 
`Task::scheduled`, `DataObject::scheduled`, `Worker::scheduled_tasks`, 
`Worker::scheduled_objects`, both in positive and negative sense (to assign or unassign).

The executor assigns/unassigns the tasks/objects depending on this scheduler plan. The 
mechanism for the scheduler to signal the changed objects is TBD. (Currently scan all.)

In every task/worker/session/client/object, the scheduler keeps any internal metadata 
in `T::sched`, and in `Graph::sched`.
 `` 

### Task states

| S | R | W | State       |T.assigned|W.assigned|T.sched.|W.sched. |W.sched_ready| 
|---|---|---|-------------|----------|----------|--------|---------|-------------|
|   |   |   | NotAssigned |          |          |        |         |             |
| x |   |   | NotAssigned |          |          | W      | x       |             |
|   | x |   | Ready       |          |          |        |         |             |
| x | x |   | Ready       |          |          | W      | x       | x           |
| x | x | x | Assigned    | W        | x        | W      | x       |             |
| x | x | x | Running     | W        | x        | W      | x       |             |
| x | x |   | Finished    |          |          |        |         |             |

Logical properties:
* S - the scheduler has assigned the task to a worker
* R - all the prerequisites of the task are ready
* W - worker knows about the task

### Data object states

|S0 |W0 |T.R|T.F|S1 |W1A|W1L|State     |O.sched. |O.assigned|O.located|   |
|---|---|---|---|---|---|---|----------|---------|----------|---------|---|
|   |   |   |   |   |   |   |Unfinished|         |          |         |   |
| x |   |   |   |   |   |   |Unfinished| W0      |          |         |   |
|   |   | x |   |   |   |   |Unfinished|         |          |         |   |
| x |   | x |   |   |   |   |Unfinished| W0      |          |         |   |
| x | x | x |   |   |   |   |Unfinished| W0      | W0       |         |   |
| x | x | x | x |   |   |   |Finished  | W0      | W0       | W0      |   |
| x | x | x | x | x |   |   |Finished  | W0,W1   | W0       | W0      |   |
| x | x | x | x | x | x |   |Finished  | W0,W1   | W0,W1    | W0      |   |
| x | x | x | x | x | x | x |Finished  | W0,W1   | W0,W1    | W0,W1   |   |
| x | x | x | x |   | x | x |Finished  | W0      | W0,W1    | W0,W1   |   |
| x | x | x | x |   |   | x |Finished  | W0      | W0       | W0 (!)  |   |
|   |   | x | x |   |   |   |Removed   |         |          |         |   |

Logical properties:
* S0 - producer has been scheduled
* W0 - worker with producer task knows about the object
* T.R - all the producer task inputs are satisfied
* T.F - producer task has finished
* S1 - the scheduler has assigned the task to another worker W*
* W1A - worker W1 has been instructed to download a copy
* W1L - worker W1 has a full copy of the object
* (!) - the server does not wait for (or get) remove confirmation wrom workers

For streams, this is sightly different: The producing worker must be notified about all
 consumers and wait for their pulls. There are no `Finished` states or presence on 
 multiple workers. We may want to indicate the stream progress (?). 