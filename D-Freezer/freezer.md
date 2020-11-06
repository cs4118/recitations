# Implementing a scheduling class

## Introduction
One of the reasons students struggle with this assignment boils down to a lack of understanding of how the scheduler works. In this guide, I hope to provide you with a clear understanding of how the Linux scheduler pieces fit together. I hope to paint a picture that you can use to help you implement the freezer scheduler.

## The `task_struct`

Recall from HW1 that in Linux, every process is defined by its `struct task_struct`. When you have multiple tasks forked off a common parent, they are linked together in a doubly linked-list `struct list_head siblings` embedded within the `task_struct`. For example, if you had four processes running on your system, each forked off one parent, it will look something like this (the parent is not shown): 

<div align='center'>
    <img src='./task_struct.png'/><br/>
</div>

However, at this stage, none of these processes are actually running on a CPU. In order to get them onto a CPU, I need to introduce you to the `struct rq`. 

## The `struct_rq`

The `struct rq` is a per-cpu run queue data structure. I like to think of it as the virtual CPU. It contains a lot of information (must of which goes way over my head), but it also includes the list of tasks that will (eventually) run on that CPU. 
So how does that work? 
A naive implementation would be to embed a `struct list_head runqueue_head` (for example) into the `struct rq`, and embed a `struct list_head node` into every `task_struct`. 
<div align='center'>
    <img src='./naive.png'/><br/>
    This is a BAD implementation. 
</div>
The main problem with this implementation is that it does not extend well. At this point, you know Linux has more than one scheduling class. Linux comes built with a Deadline class, a Real-time class, and the primary CFS. Having a `list_head` embedded directly into the `struct rq` for each scheduling class is not feasible. The solution is to create a new structure containing the `list_head` and any bookkeeping variables into a separate structure. Then, we can include just the wrapper structure in the `struct rq`. Linux includes these structures in `linux/kernel/sched/sched.h`. 

By convention, Linux names these `struct {sched class}_rq`. For example, the CFS class is called `struct cfs_rq` and `struct cfs_rq cfs` for the strcture definition and member of `struct rq` respectivelly. 

## The `freezer_rq`

At this point, you have probably guessed that you will need to do the same thing for the freezer. You are right. The `feezer_rq` should include the head of the freezer runqueue. Additionally, you may need to include some bookkeeping variables. Word of advice, think of what you would actually need and don't have anything extra (it should be pretty simple). 

## The `sched_freezer_entity`

Now that you have the `struct rq` setup, you need to have some mechanism to join your `task_struct`s into the queue. Here, too, you can't just include a `list_head node` to add a task onto the runqueue, but you can't include the `struct frezzer_rq` either for the simple reason that the bookkeeping is different. As you have probably guessed, we are going to wrap the list_head and all the bookkeeping variables in its own struct. In Linux, we name these structs `sched_{class}_entity` (one exception is that CFS names this `sched_entity`). For example, the Real-time scheduling class calls it `sched_rt_entity`. We will name ours `struct sched_freezer_entity`. Again, make sure you only include what you need in this struct. 

With all this setup, here is what the final picture would look like this.


<div align='center'>
    <img src='./freezer.png'/><br/>
</div>
