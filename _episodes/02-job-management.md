---
title: "Job Submission and Management"
slug: "dirac-job-scheduling-job-management"
teaching: 0
exercises: 0
questions:
- "How do I request specific resources to use for a job?"
- "What is the life cycle of a job?"
- "What can I do to specify how my job will run?"
- "How can I find out the status of my running or completed jobs?"
objectives:
- "Specify a job's resource requirements in a Slurm batch script."
- "Describe the main states a job passes through from start to completion."
- "Cancel multiple jobs using a single command."
- "Describe how backfilling works and why it's useful."
- "Obtain greater detail regarding the status of a completed job and its use of resources."
keypoints:
- "Use `--nodes` and `--ntasks` in a Slurm batch script to request the total number of machines and CPU cores you'd like for your job."
- "A typical job passes through pending, running, completing, and completed job states."
- "Reasons for a job's failure include being out of memory, being suspended, cancelled, or exiting with a non-zero exit code."
- "Determinig backfill windows allows us to determine when we may make use of idle resources."
- "Use `scontrol` to present detailed information concerning a submitted job."
- "Use `sstat` to query the used resources for an actively running job."
- "Use `sacct` to query the resource accounting information for a completed job."
---

We've recapped the fundamentals of how we are able to use Slurm to submit a simple job and monitor it to completion,
but in practice we'll need more flexibility in specifying what resources we need for our jobs,
how they should run, and how we manage them, so let's look at that in more detail now.

## Selecting Resources for Jobs

### Selecting Available Resources

We saw earlier how to use `sinfo -s` as a means to obtain a list of queues we are able to access,
but there's also some additional useful information.
Of particular interest is the `NODES` column, which gives us an overview of the state of these resources, and hence
allows us to select a queue with those resources that are sufficiently available for jobs we'd like to submit. For example, on Tursa you may see something like:

~~~
PARTITION AVAIL  TIMELIMIT   NODES(A/I/O/T) NODELIST
slurm*       up 4-00:00:00      198/0/0/198 dnode[001-198]
devel        up    2:00:00      198/2/0/200 dnode[001-200]
~~~
{: .output}

Looking at the `NODES` column, it indicates how many nodes are:

- `A`ctive - these are running jobs
- `I`dle - no jobs are running
- `O`ther - these nodes are down, or otherwise unavailable
- The `T`otal number of nodes

The `NODELIST` is a summary of those nodes in a particular queue.

In this particular instance, we can see that 2 nodes are idle in the `devel` queue,
so if that queue fits our needs (and we have access to it) we may decide to submit to that.

### Specifying Job Resource Requirements

One thing that is absolutely critical when working on an HPC system
is specifying the resources required to run a job, which allows the scheduler to
find the right time and place to schedule our job. If you do not specify
requirements (such as the amount of time you need), you will likely be stuck
with your site's default resources, which is probably not what you want.

When launching a job, we can specify the resources our job needs in our job script,
using `#SBATCH <parameter>`. We've used this format before when indicating the account and queue to use.
You may have seen some of these parameters before, but let's take a look at the most important
ones and how they relate to each other.

For these parameters, by default a *task* refers to a single CPU unless otherwise indicated.

- `--nodes` - the total number of machines or nodes to request
- `--ntasks` - the total number of CPU cores (across requested nodes) your job needs. Generally, this will be 1 unless you're running MPI jobs which are
able to use multiple CPU cores. In which case, it essentially specifies the number of MPI ranks to start across the nodes. For example, if `--nodes=4` and `--ntasks=8`, we're requesting 4 nodes, with each node having 2 CPU cores, since to satisfy the requirement of a total of 8 CPU cores over 4 nodes, each node would need 2 cores. So here, Slurm does this calculation for us. We'll see an example of using `--ntasks` with MPI in a later episode.

> ## Being Specific
>
> Write and submit a job script - perhaps adapted from our previous one - called `multi-node-job.sh` that requests a total of 2 nodes with 2 CPUs per node.
> Remember to include any site-specific `#SBATCH` parameters.
>
> > ## Solution
> >
> > ~~~
> > #!/bin/bash -l
> > #SBATCH --account=yourAccount
> > #SBATCH --partition=aPartition
> > #SBATCH --nodes=2
> > #SBATCH --ntasks=4
> > #SBATCH --time=00:00:30
> >
> > echo "This script is running on ... "
> > hostname
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > sbatch multi-node-job.sh
> > ...
> > squeue -u yourUsername
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> >   JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
> > 6485195 cosma7-pa mul-job. yourUse+  R       0:05      2 m[7400-7412]
> > ~~~
> > {: .output}
> > 
> > Here we can see that the job is using a total of two nodes as we'd like - `m7400` and `m7401`.
>{: .solution}
{: .challenge}

Our example earlier with `--nodes=4` and `--ntasks=8` meant Slurm calculated that 2 cores would be needed on each of the four nodes (i.e. `ntasks/nodes = 2 cores per node`). So the number of cores needed was implicit.
However, we can also specify the number of CPU cores per node explicitly using `--ntasks-per-node`.
In this case, we use this with `--nodes` and we don't need to specify the total number of tasks with `--ntasks` at all.
So using our above example with `--nodes=4`, to get our desired total 8 CPU cores we'd specify `--ntasks-per-node=2`.

> ## Being Even More Specific
>
> Write and submit a job script that uses `--nodes` and `--ntasks-per-node` to request a total of 2 nodes with 2 CPUs
> per node.
>
> > ## Solution
> >
> > ~~~
> > #!/bin/bash -l
> > #SBATCH --account=yourAccount
> > #SBATCH --partition=aPartition
> > #SBATCH --nodes=2
> > #SBATCH --ntasks-per-node=2
> > #SBATCH --time=00:00:30
> >
> > echo "This script is running on ... "
> > hostname
> > ~~~
> > {: .language-bash}
> > 
> > Once submitted, using `squeue` we should see the same results as before, again using a total of two nodes.
>{: .solution}
{: .challenge}

We'll be looking at how we can make full use of these parameters with MPI jobs later in this lesson.

Up until now, the number of tasks have been synonymous with the number of CPUs. However, we can also specify a multiple
number of CPUs per task too, by using `--cpus-per-task` - so if your job is multithreaded, for example it makes use of
OpenMP, you can specify how many threads it needs using this parameter.

> ## What about Parameters Named -A, -N, -J and others?
> 
> You may also see some other shorter parameter names. In Slurm, some `SBATCH` options also have shortened forms, but
> mean the same thing. For example:
> 
> |**Long form**       | **Short form**     |
> |--------------------|--------------------|
> |`--job-name`        | `-J`               |
> |`--nodes`           | `-N`               |
> |`--ntasks`          | `-n`               |
> |`--cpus-per-task`   | `-c`               |
> 
> Not all parameters have a short form, for example `--nodes-per-task` and `--exclusive`.
>
> In practice you can use either, although using the longer form is more verbose and helps remove ambiguity,
> particularly for newer users.
{: .callout}

> ## Only Request What you Require
> 
> When specifying resources your job will need it's important not to ask for too much.
> Firstly, because any resources you request but don't use (e.g. CPUs, memory, GPUs) will be wasted
> and potentially cost more in terms of your account usage, but also because requesting larger resources
> will take longer to queue. It also means that other user's jobs that would have been a better fit for these
> resources may also take longer to run. It's good to remember that we are part of a wider community of HPC users, and
> as with any shared resource, we should act responsibly when using it.
> 
> As we'll see later, using the `sacct` command we're able to find out what resources previous jobs actually used, and
> then optimise the resources we request in future job runs.
{: .callout}

## Managing Job Submissions

### Monitoring a Job

As we've seen, we can check on our job's status by using the command `squeue`. Let's take a look in more detail.

~~~
squeue -u yourUsername
~~~
{: .language-bash}

You may find it looks like this:

~~~
  JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
5791510 cosma7-pa example- yourUser PD       0:00      1 (Priority)
~~~
{: .output}

So `-u yourUsername` shows us all jobs associated with our machine account. We can also use `-j` to query specific
job IDs, e.g.: `squeue -j 5791510` which will, in this case, yield the same information since we only have that job
in the queue (if it hasn't already completed!).

In either case, we can see all the details of our job, including the partition, user, and also the state of the job (in the `ST` column).
In this case, we can see it is in the PD or PENDING state.
Typically, a successful job run will go through the following states:

- `PD` - *pending*: sometimes our jobs might need to wait in a queue first before they can be allocated to a node to run. 
- `R` - *running*: job has an allocation and is currently running
- `CG` - *completing*: job is in the process of completing
- `CD` - *completed*: the job is completed

For pending jobs, helpfully, you may see a reason for this in the `NODELIST(REASON)` column;
for example, that the nodes required for the job are down or reserved for other jobs,
or that the job is queued behind a higher priority job that is making use of the requested resources.
Once it's able to run, the nodes that have been allocated will be displayed in this column instead.

However, in terms of job states, there are a number of reasons why jobs may end due to a failure or other condition, including:

- `OOM` - *ouf of memory*: the job attempted to use more memory during execution than what was available
- `S` - *suspended*: job has an allocation, but it has been suspended to allow for other jobs to use the resources
- `CA` - *cancelled*: the job was explicitly cancelled, either by the user or system administrator, and may or may not have been started
- `F` - *failed*: the job has terminated with a non-zero exit code or has failed for another reason

You can get a full list of job status codes via the [SLURM documentation](https://slurm.schedmd.com/squeue.html#lbAG).

### Cancelling a Job

Sometimes we'll make a mistake and need to cancel a job. This can be done with
the `scancel` command. Let's submit a job and then cancel it using
its job number (remember to change the walltime so that it runs long enough for
you to cancel it before it is killed!).

~~~
sbatch example-job.sh
squeue -u yourUsername
~~~
{: .language-bash}

~~~
Submitted batch job 5791551

  JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
5791551 cosma7-pa hello-wo yourUser PD       0:00      1 (Priority)
~~~
{: .output}

Now cancel the job with its job number (printed in your terminal). A clean
return of your command prompt indicates that the request to cancel the job was
successful.

~~~
scancel 5791551
# It might take a minute for the job to disappear from the queue...
squeue -u yourUsername
~~~
{: .language-bash}

~~~
...(no output when there are no jobs to display)...
~~~
{: .output}

> ## Cancelling multiple jobs
>
> We can also cancel all of our jobs at once using the `-u yourUsername` option. This will delete all jobs for a specific user (in this case, yourself). Note that you can only delete your own jobs.
>
> Try submitting multiple jobs and then cancelling them all.
>
> > ## Solution
> >
> > First, submit a trio of jobs:
> >
> > ~~~
> > sbatch example-job.sh
> > sbatch example-job.sh
> > sbatch example-job.sh
> > ~~~
> > {: .language-bash}
> >
> > Then, cancel them all:
> >
> > ~~~
> > scancel -u yourUsername
> > ~~~
> > {: .language-bash}
>{: .solution}
{: .challenge}

### Backfilling

When developing code or testing configurations it is usually the case that you don’t need a lot of time.
When that is true and the queues are busy, backfilling is a scheduling feature that proves really useful.

If there are idle nodes then that means they are available to run jobs, or that they are being kept so that a
job can run in the future. The time between when a job needs those nodes and now is the backfill window
and jobs that need less than that time may be scheduled to run on those resources. To take best advantage of backfilling, 
specify a shorter time period for jobs so they can fill these windows.

You can check if a particular Slurm scheduler is configured to use the backfill technique (which is the default) by doing the following:

~~~
scontrol show config | grep SchedulerType
~~~
{: .language-bash}

This use of the `scontrol` command allows us to see the configuration of Slurm, and by piping its output (using `|`) to
the `grep` command, we can search for a particular configuration field called `SchedulerType` which holds the method used
for scheduling. If a backfill scheduler is being used, you should see the following:

~~~
SchedulerType           = sched/backfill
~~~
{: .output}

If the `builtin` scheduler is used instead, then jobs will be executed in order of submission, regardless if there
are any free resources to run your job immediately.

> ## DiRAC COSMA: Handy Backfill Tools
> 
> On DiRAC's COSMA you are able to see backfill availability windows for various queues
> (e.g. for COSMA5, COSMA7,and COSMA8) by using a particular backfill command. For example, for COSMA5:
> 
> ~~~
> c5backfill
> ~~~
> {: .language-bash}
> 
> ~~~
> # Backfill windows available for cosma queues:
> #
> No. hosts/slots:  1 / 16
> Runtime:          31 hours 49 minutes 26 seconds
> Hostnames:        m5336
> 
> No. hosts/slots:  5 / 80
> Runtime:          72 hours 00 minutes 00 seconds
> Hostnames:        m[5185,5315,5337,5366,5377]
> ~~~
> {: .output}
>
> So here, we can see that on COSMA5 we have only one node available for 31 hours and 49 minutes, and 5 nodes available
> for 72 hours.
{: .callout}

### Inspecting and Controlling the State of Jobs

So far we've used the `squeue` command to check the general status of pending and running jobs. But we can also obtain
even more information using the `scontrol` command regarding the job's status, configuration and use of resources.

Let's create a new job to test this, that uses a single node and a single CPU on that node to run a single basic task:

~~~
#!/usr/bin/env bash
#SBATCH --account=yourAccount
#SBATCH --partition=aPartition
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=1
#SBATCH --ntasks=1
#SBATCH --time=00:00:30
#SBATCH --job-name=specific-job

sleep 25
echo "This script is running on ... "
hostname
~~~
{: .language-bash}

Next, launch our new job:

~~~
sbatch specific-job.sh
~~~
{: .language-bash}

...then make a note of the job ID returned, e.g.:

~~~
...
Submitted batch job 309281
~~~
{: .output}

We can then use this job ID to ask Slurm for more information about it:

~~~
scontrol show jobid=309281
~~~
{: .language-bash}

On DiAL3, it would look something like this (although of course, on other sites it will differ in parts):

~~~
JobId=309281 JobName=specific-job
   UserId=dc-crou1(210462) GroupId=dirac(59019) MCS_label=N/A
   Priority=7603 Nice=0 Account=ds007 QOS=normal
   JobState=COMPLETED Reason=None Dependency=(null)
   Requeue=1 Restarts=0 BatchFlag=1 Reboot=0 ExitCode=0:0
   RunTime=00:00:25 TimeLimit=00:01:00 TimeMin=N/A
   SubmitTime=2024-08-01T11:57:18 EligibleTime=2024-08-01T11:57:18
   AccrueTime=2024-08-01T11:57:18
   StartTime=2024-08-01T11:57:19 EndTime=2024-08-01T11:57:44 Deadline=N/A
   SuspendTime=None SecsPreSuspend=0 LastSchedEval=2024-08-01T11:57:19 Scheduler=Main
   Partition=devel AllocNode:Sid=d3-login01:2159102
   ReqNodeList=(null) ExcNodeList=(null)
   NodeList=dnode036
   BatchHost=dnode036
   NumNodes=1 NumCPUs=128 NumTasks=1 CPUs/Task=1 ReqB:S:C:T=0:0:*:*
   TRES=cpu=128,node=1,billing=128
   Socks/Node=* NtasksPerN:B:S:C=1:0:*:* CoreSpec=*
   MinCPUsNode=1 MinMemoryNode=0 MinTmpDiskNode=0
   Features=(null) DelayBoot=00:00:00
   OverSubscribe=NO Contiguous=0 Licenses=(null) Network=(null)
   Command=/lustre/dirac3/home/dc-crou1/job-sched-testing/specific-job.sh
   WorkDir=/lustre/dirac3/home/dc-crou1/job-sched-testing
   StdErr=/lustre/dirac3/home/dc-crou1/job-sched-testing/slurm-309281.out
   StdIn=/dev/null
   StdOut=/lustre/dirac3/home/dc-crou1/job-sched-testing/slurm-309281.out
   Power=
~~~
{: .output}

In particular, we can see:

- As we might expect the `JobState` is `RUNNING`, although it may be `PENDING` if waiting to be assigned to a node, or if we weren't fast enough running the `scontrol` command it might be `COMPLETED`
- How long the job has run for (`RunTime`), and the job's maximum specified duration (`TimeLimit`)
- The job's `SubmitTime`, as well as the job's `StartTime` for execution: this may be the actual start time, or the expected start time if set in the future. The expected `EndTime` is also specified, although if it isn't specified directly in the job script this isn't always exactly `StartTime` + specified duration; it's often rounded up, perhaps to the nearest minute.
- The queue assigned for the job is the `devel` queue, and that the job is running on the `dnode036` node
- The resources assigned to the job are a single node (`NumNodes=1`) with 128 CPU cores, for a single task with 1 CPU core per task. Note that in this case we got more resources in terms of CPUs than what we asked for. For example in this instance, we can see that we actually obtained a node with 128 CPUs (although we won't use them)
- We didn't specify a working directory within which to execute the job, so the default `WorkDir` is our home directory
- The error and output file locations, as specified by `StdErr` and `StdOut`

You can find more information on `scontrol` and the various fields in the [Slurm documentation](https://slurm.schedmd.com/scontrol.html).

Note that typing `scontrol` on its own will enter an interactive state with an `scontrol` prompt which allows
you to enter subcommands (like `show` or `update`) directly to the `scontrol` command. You can exit by typing `quit`.

> ## Let's Wait a Bit...
> 
> It may be useful to defer the scheduling of a job until some point in the future.
> We can specify a delayed start time (or even delayed start date) by using the `--begin` parameter in our job script.
> For example, adding `#SBATCH --begin=16:00` will delay starting the job until 16:00 (assuming there is an appropriate
> node available at that time). We can also specify a relative delay time, for example passing `now+1hour` instead.
> 
> Launch the job again, but this time specify a start time of 1 minute in the future in the job script. How does the
> `scontrol` output change for a delayed job?
> 
> > ## Solution
> >
> > First add `#SBATCH --begin=now+1minute` to the parameters in the job script, and relaunch using `sbatch`.
> >
> > You should see in the job's `scontrol` output:
> >
> > - `JobState=PENDING`, `RunTime=00:00:00`, and `AccrueTime` is `Unknown`, since it's waiting to run and no runtime has yet elapsed
> > - `EligibleTime` will specify the calculated start time one minute into the future
>{: .solution}
{: .challenge}

## Querying Job Resources

For more detail about jobs and the resources, there are two key Slurm commands we can use to give us more information about jobs
that are either actively running (using `sstat`) or completed (using `sacct`).

### Querying Running Jobs

First, submit our previous job script `specific-job.sh` using `sbatch`,
then check when it's actually running using `squeue` (since `sstat` will only work on actively running jobs),
and then when we know it's running we can use `sstat` to query which resources it's using.
Note that it's probably a good idea to increase the `25` value for `sleep` in the job script
(along with the `#SBATCH --time` parameter) so you have more time to query the job before it completes!

For example, following submission via `sbatch`:

~~~
squeue -j 6803898
~~~
{: .language-bash}

Wait until it reaches the running state designated by `R`:

~~~
  JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
6803898 cosma7-pa hello-wo yourUser  R       0:00      1 (Priority)
~~~
{: .output}

Since we now know it's running, we may query its active resource usage to date.

~~~
sstat -j 6803898
~~~
{: .language-bash}

The first line of output is presented as a series of columns representing the various properties - or fields - associated with the job,
but unfortunately you'll notice that with so many columns it renders the output quite unreadable!
Fortunately we can select which columns we wish to see, e.g.:

~~~
sstat -j 6803898 --format=JobID,AveCPU,NTasks,MinCPU,MaxDiskWrite
~~~
{: .language-bash}

So here, we've selected a number of field columns we're interested in:

- `JobID` - the ID of the job, useful if we happen to specify multiple jobs to query, e.g. `-j 6803898,6803899,6803900`.
- `AveCPU` - the average CPU time of the job
- `NTasks` - total number of tasks in a job
- `MinCPU` - the minimum CPU time used in a job
- `MaxDiskWrite` - the maximum number of bytes written by the job

But looking at the output, you'll notice there is no information is returned other than the field headers:

~~~
JobID            AveCPU   NTasks     MinCPU MaxDiskWrite
------------ ---------- -------- ---------- ------------ 
~~~
{: .output}

This is because, by default, all jobs are actually made up of a number of job *steps*,
and batch scripts running by themselves aren't accounted for.

A job that consists of only a batch script, like this one, typically only has two steps:
 - *batch step*, which is created for all jobs submitted with `sbatch`
 - *extern* step, which accounts for resources used by a job outside of aspects managed by Slurm
(which includes special cases such as SSH-ing into the compute node, which may be allowed on some systems
and would be accounted here)

A job may also have additional steps,
for example with MPI jobs there will be extra processes launched which are accounted for as separate steps,
and will be listed separately.
In our case, all processing is accounted for in the `batch` step, so we need to query that directly,
since by default, `sstat` doesn't include this.
We can reference the batch step directly using:

~~~
sstat -j 6803898.batch --format=JobID,AveCPU,NTasks,MinCPU,MaxDiskWrite
~~~
{: .language-bash}

And now we should see something like the following (you may need to submit it again):

~~~
JobID            AveCPU   NTasks     MinCPU MaxDiskWrite
------------ ---------- -------- ---------- ------------
6803898.batch  00:00:00        1   00:00:00           66
~~~
{: .output}

In this trivial case, so far we can see that we've used virtually no actual CPU,
and the batch submission script has written a total of 66 bytes.

There's also the `--allsteps` flag to see a summary of all
steps, although note that this isn't always supported on all Slurm systems:

~~~
sstat --allsteps -j 6803898 --format=JobID,AveCPU,NTasks,MinCPU,MaxDiskWrite
~~~
{: .language-bash}

So in our batch case, we'd see two steps:

~~~
JobID             AveCPU   NTasks     MinCPU MaxDiskWrite 
------------- ---------- -------- ---------- ------------ 
6803898.exte+   00:00:00        1   00:00:00        1.00M 
6803898.batch   00:00:00        1   00:00:00           66
~~~
{: .output}


> ## What About other Format Fields?
> 
> We've looked at some example fields to include in the output format,
> but there are *many* others that you may wish to consider,
> which you can find in the [Slurm documentation](https://slurm.schedmd.com/sstat.html#SECTION_Job-Status-Fields).
> 
> Another way to get help on the field formats is to use `sstat --helpformat`,
> which handily gives you a list of all fields you can use.
{: .callout}

### Querying Completed Jobs

Once a job is complete, since it only operates on active jobs you'll notice that `sstat` no longer presents resource usage.
For jobs that have completed, we use `sacct`, which displays accounting data for Slurm jobs.

We can first use `sacct` to list the completed jobs for today:

~~~
JobID           JobName  Partition    Account  AllocCPUS      State ExitCode 
------------ ---------- ---------- ---------- ---------- ---------- -------- 
6803854            bash cosma7-pa+ yourAccou+         28    TIMEOUT      0:0 
6803854.ext+     extern            yourAccou+         28  COMPLETED      0:0 
6803854.0          bash            yourAccou+         28  COMPLETED      0:0 
6803898      specific-+ cosma7-pa+ yourAccou+         28  COMPLETED      0:0 
6803898.bat+      batch            yourAccou+         28  COMPLETED      0:0 
6803898.ext+     extern            yourAccou+         28  COMPLETED      0:0 
6804261_1    hello_wor+ cosma7-pa+ yourAccou+         28  COMPLETED      0:0 
6804261_1.b+      batch            yourAccou+         28  COMPLETED      0:0 
6804261_1.e+     extern            yourAccou+         28  COMPLETED      0:0 
6804261_2    hello_wor+ cosma7-pa+ yourAccou+         28  COMPLETED      0:0 
6804261_2.b+      batch            yourAccou+         28  COMPLETED      0:0 
6804261_2.e+     extern            yourAccou+         28  COMPLETED      0:0 
6804261_3    hello_wor+ cosma7-pa+ yourAccou+         28  COMPLETED      0:0 
6804261_3.b+      batch            yourAccou+         28  COMPLETED      0:0 
6804261_3.e+     extern            yourAccou+         28  COMPLETED      0:0 
~~~
{: .output}

Or, for a specific job:

~~~
sacct -j 6803898
~~~
{: .language-bash}

Which will display accounting information for a given job, including its constituent steps
(and we don't need to use `--allsteps` to collate this information).

By default, we'll see something like the following:

~~~
JobID           JobName  Partition    Account  AllocCPUS      State ExitCode 
------------ ---------- ---------- ---------- ---------- ---------- -------- 
6803898      specific-+ cosma7-pa+ yourAccou+         28  COMPLETED      0:0 
6803898.bat+      batch            yourAccou+         28  COMPLETED      0:0 
6803898.ext+     extern            yourAccou+         28  COMPLETED      0:0 
~~~
{: .output}

As with `sstat`, we are also able to customise the fields we wish to see, e.g.:

~~~
sacct -j 6803898 --format=JobID,JobName,Partition,Account,AllocCPUS,State,Elapsed,CPUTime
~~~
{: .language-bash}

~~~
JobID           JobName  Partition    Account  AllocCPUS      State    Elapsed    CPUTime 
------------ ---------- ---------- ---------- ---------- ---------- ---------- ---------- 
6803898      specific-+ cosma7-pa+ yourAccou+         28  COMPLETED   00:00:25   00:11:40 
6803898.bat+      batch            yourAccou+         28  COMPLETED   00:00:25   00:11:40 
6803898.ext+     extern            yourAccou+         28  COMPLETED   00:00:25   00:11:40
~~~
{: .callout}

As with `sstat`, you can add many [other fields](https://slurm.schedmd.com/sacct.html#SECTION_Job-Accounting-Fields) too,
although note the accounting data presented for these will be different depending on the HPC system's configuration and the job's type.
CPUTime here is equal to `AllocCPUs * Elapsed`, i.e. the number of CPUs allocated to the job multiplied by the total elapsed time.

> ## So Much to Remember!
> 
> Slurm has a great many commands - as well as command arguments - and it can prove difficult
> to remember them all. It's often helpful to make a note of those you commonly use,
> or make use of a pre-made reference/cheat sheet such as
> [this one](https://slurm.schedmd.com/pdfs/summary.pdf) which has a comprehensive list of commands
> and and their arguments with helpful short descriptions.
{: .callout}
