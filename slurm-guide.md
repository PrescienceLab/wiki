# Slurm Guide

Welcome to the Slurm guide!
This quick guide will walk you through using Slurm in the cheese cluster.

## Why are we using Slurm?
We have a few reasons:
  * Some people need to fire off thousands or millions of tasks/jobs that consume very little resources, but we want to be fair other users on the machine.
  * Some people need very quiet machines to get accurate performance numbers.
  * We need to (scalably) arbitrate allocations to scarce resources (GPUs, FPGAs, etc.)

NOTE: We are NOT using Slurm the way it is used in supercomputers!
We are ***NOT*** removing your ability to interactively log-in and use the machines that we have enrolled in Slurm!

NOTE: By default, if you submit a job to Slurm, it will run on one of the x86 machines we have in the cluster.
This currently includes:
  * dubliner
  * roquefort
  * limburger
  * jarlsberg
  * manchego
  * v-test-phi[0..3]

We have also enrolled Burrata in the cluster, but your jobs will ***NOT*** be sent to burrata by default.
We did this to give people fewer surprises when developing their software.
Please ask one of the Cheese administrators (Nick, Karl, Alex, or Peter) how to access Burrata using Slurm.

# Submitting a Job
There are 2 ways to submit jobs, that correspond to "the 2 ways" to use a computer:
  1. Batch/Async/Background
  2. Interactive/Sync

There is **exactly** one command for each of these:
  1. `sbatch <job-script>`
  2. `salloc`

Any job you submit will run ***as your user***!
Mind that the only storage shared across all nodes in `/tank` and when your job is running (either in batch or interactive mode) you are working with that node's ***local*** storage.
This is another good reason to leverage `/tank`!

Since different nodes may have different software installed, we recommend you make use of Nix and Nix flakes to pull in your project dependencies when possible.
If you do not want to use Nix, you can force your jobs to run on the same node you are using for development (see "Useful Flags" below).

## `sbatch`
`sbatch` lets you submit a "job-script" to Slurm for execution.
Slurm will schedule your job to execute when the resources you requested become available.

NOTE: You **MUST** give `sbatch` something that starts with a shebang line (`#!/usr/bin/env bash` for instance).

Below is an example on how to use `sbatch`:

```sh
$ cat example-job.sh
#!/usr/bin/env bash

echo "Hello to the world, from Slurm!"
exit 0

$ sbatch ./example-job.sh
```

### `sbatch` Comment-Directives
You can use command-line flags to configure `sbatch` submissions.
But if you have set of flags you want to reuse over and over again, consider using a comment-directive.
Below is an example job-script that shows how to use comment-directives.

```bash
#!/usr/bin/env bash

# Normal comment

# Below is an example of a comment-directive
#SBATCH --exclusive
#SBATCH --nodes=1
#SBATCH --job-name='my-special-job'

# You can comment-out a comment-directive easily:
##SBATCH --mem=3K

# NOTE: Comment-directives will only be applied BEFORE any command in the shell script!
set -o errexit
```

## `salloc`
Using `salloc` on any of the cheese machines will grant you an allocation on one of the other x86 cheese machines by default.
Exactly which one depends on what everybody else is doing.

Unlike Slurm instances you have used before, you do **NOT** need to request an allocation then open a shell.
We do this for you by default.
If you just type `salloc`, you will be granted an allocation on a machine and your shell will be redirected to that machine, just like an SSH connection.
To end your interactive allocation, you can use the normal `exit` command or press Ctrl+d.

An example of `salloc`'s use and output is shown below:

```console
karl@dubliner:~$ salloc
salloc: Granted job allocation 717
groups: cannot find name for group ID 125

karl@manchego:~$ <Ctrl+d>
exit
salloc: Relinquishing job allocation 717
```

Perhaps you only need 8 cores for your job.
You can use the `-c`/`--cpus-per-task` flag to provide that information.
You can use the `nproc` command to determine how many cores you have once you are given an allocation.

```console
karl@dubliner:~$ salloc -c 8
salloc: Granted job allocation 750

karl@roquefort:~$ nproc
8
karl@roquefort:~$ exit
salloc: Relinquishing job allocation 750
```

## Getting a Job Allocation, but not Opening a Shell
Sometimes you want to request a job allocation but not immediately switch your shell to the remote machine.
Use the `--no-shell` flag to achieve this behavior.

When you want to run job steps on your allocation you must submit the job step using `srun`.
This is the same behavior as when you use job steps in `sbatch`.
Note that you can submit multiple job steps against this allocation at a time.
You can also optionally allow the job steps to overlap if you want using the `--overlap` flag.
If you want an interactive shell on the allocation, you need to use `--pty` flag to connect your terminal to the Slurm input/output.

There is NO way to detach from an interactive allocation easily.
You need to use another tool, like tmux, to achieve this behavior.
In summary, you CAN attach to an allocation after requesting it, but you CANNOT detach from an allocation that was immediately opened as interactive (an allocation that immediately opens a shell).

Below is an example of requesting an allocation for an indefinite amount of time then delaying opening a shell until a later time.
```console
karl@dubliner:~$ salloc --no-shell
salloc: Granted job allocation 1389

karl@dubliner:~$ srun --pty --overlap --jobid 1389 bash

karl@roquefort:~$
```

If you are done with your allocation before your time limit, then you must cancel your job using `scancel`.

```console
# Finished with roquefort. Since this was an indefinite allocation, I must
# manually cancel it

karl@dubliner:~$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
              1389   compute no-shell     karl  R      12:29      1 roquefort

karl@dubliner:~$ scancel 1389

karl@dubliner:~$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
```

# General Recommendations
Here are some recommendations from us, the admins, about how you could use Slurm:
  * `#SBATCH` comment-directives in your submitted script ***MUST*** come before any shell commands!
    Comments before and after `#SBATCH` comment-directives are allowed, but Slurm stops looking for them as soon as the first non-comment shell command is found.
  * Make sure your scripts record some unique component when writing outputs.
    Slurm does this automatically with `STDOUT` on your behalf.
    But if you create files/directories, then Slurm will not handle that for you and you must manually uniquify them.
    We recommend either a timestamp or the job ID, as those will be quite unique over time.
  * Try to make your script that runs inside the job fairly idempotent.
    For example, it can assume it is running inside of Slurm, but it should not assume that the directory it is working on is your working copy.
    Instead, make the working script assume a fresh copy of your work, and that one of the jobs of your Slurm job is to compile your test programs.
    Obviously, this is not tenable for enormous compile jobs.
    Use your best judgment.
  * Take advantage of the `--chdir` flag on `sbatch` to change the working directory of the Slurm job.
    You can build a submission directory that your job should work out of, which helps keep your working files separate from your job files.
  * For very long `sbatch` runs, make use of `srun` to split the job up into steps.
    Slurm can restart jobs that fail for a variety of reasons (your node went down, the server crashed, etc.), but only at the granularity of a step.

# Useful Flags
Here are a quick run-down of useful flags you can pass to some Slurm commands.
Please check the documentation for each command (`man sbatch` for instance), as some flags may not be present on certain commands or behave slightly different.
In addition, there are ***MANY*** more flags than the ones we present below; check the documentation for them all.

  * `--exclusive`: Take exclusive control over the machine, even if you requested fewer resources than the whole machine has.
  * `-w`/`--nodelist=<node-list>`: Run on a specific host
  * `-c`/`--cpus-per-task=<ncpu>`: Give your job the specified number of hyperthreads/logical cores.
    NOTE: If you choose an odd number, it will be rounded up to give you a full physical CPU.
    Your assigned hyperthreads will **NOT** overlap with another persons'/jobs'.
  * `--mem=<size>[units]`: Give your job the specified ***TOTAL*** amount of memory.
    Size is just a number, units is ONE of [K, M, G, T].
  * `-p`/`--partition=<partition-name>`: Choose a different partition to put your job on.
    Burrata is in a different partition for instance.

A contrived example of using all of these flags is shown below:

```console
karl@dubliner:~$ sbatch -c 8 --mem 8G \
                        --partition=intel --nodelist=manchego \
                        --exclusive \
                        example-batch-file.sh
Submitted batch job 751
```

# Looking at the Queue
Use the `squeue` command to look at the current Slurm queue.

```console
karl@dubliner:~$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
               719   compute     wrap     karl  R       0:06      1 manchego
               720   compute interact     karl  R       0:02      1 manchego
               721   compute interact     nick  R       0:09      1 roquefort
```

If there are a lot of jobs in the queue, it is easy to lose where your jobs' positions.
There is a `-u`/`--user=<user1,user2>`flag to make finding *your* jobs easier:

```console
karl@dubliner:~$ echo $USER
karl
karl@dubliner:~$ squeue -u $USER
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
               719   compute     wrap     karl  R       0:06      1 manchego
               720   compute interact     karl  R       0:02      1 manchego
```

# Looking at the Cluster
We have divided the cheese machines up a little bit based on what features they have.
These include ISA (x86 vs. ARM), configuration (number of sockets), and other oddities (Xeon Phis).
The sections below show you hot to find this information.

## Listing Partitions
Partitions are a way to group multiple nodes together.
By default all of your jobs will run on the `compute` partition, which comprises a majority of our machines (note the `Default=YES` in the `compute` partition).

However, we have defined some other partitions as well.
For example, all machines with Intel CPUs are grouped under the `intel` partition.
Below is an example of how to list all partitions in the cluster:

```console
karl@dubliner:~$ scontrol show partition
PartitionName=cheese
   AllowGroups=ALL AllowAccounts=ALL AllowQos=ALL
   AllocNodes=ALL Default=NO QoS=N/A
   DefaultTime=NONE DisableRootJobs=NO ExclusiveUser=NO GraceTime=0 Hidden=NO
   MaxNodes=UNLIMITED MaxTime=UNLIMITED MinNodes=0 LLN=NO MaxCPUsPerNode=UNLIMITED
   Nodes=burrata,colbyjack,dubliner,jarlsberg,limburger,manchego,pepperjack,roquefort,string,toussaint
   PriorityJobFactor=1 PriorityTier=1 RootOnly=NO ReqResv=NO OverSubscribe=NO
   OverTimeLimit=NONE PreemptMode=OFF
   State=UP TotalCPUs=556 TotalNodes=10 SelectTypeParameters=NONE
   JobDefaults=(null)
   DefMemPerNode=UNLIMITED MaxMemPerNode=UNLIMITED

PartitionName=compute
   ...
   AllocNodes=ALL Default=YES QoS=N/A
   ...
   Nodes=dubliner,jarlsberg,limburger,manchego,roquefort
   ...

PartitionName=intel
   ...
   Nodes=dubliner,manchego
   ...
```

## Listing Nodes
Every computer in the cheese cluster that Slurm can run jobs on is a node.
Each node may have different kinds of resources, e.g. Dubliner with its GPU or Toussaint with its FPGA.
Below is an example of how to list all nodes in the cluster:

```console
karl@dubliner:~$ scontrol show node
NodeName=burrata Arch=aarch64 CoresPerSocket=128
   CPUAlloc=0 CPUTot=128 CPULoad=42.68
   ...

NodeName=dubliner Arch=x86_64 CoresPerSocket=22
   CPUAlloc=0 CPUTot=176 CPULoad=7.21
   ...

NodeName=jarlsberg Arch=x86_64 CoresPerSocket=24
   CPUAlloc=0 CPUTot=48 CPULoad=0.00
   ...
```

# Phi machines
The phi machines (v-test-phi{0..3}) have been enrolled into the cheese cave, users synced, `/tank` mounted, Nix installed (and flakes).
All 4 phis are identical software-wise, and closely match the other machines in the cluster.

You DO NOT want to develop on those machines!
They have incredibly slow CPUs, and unless you can use 4-way barrel SMT, the 256-core count versions are even slower.
BUT, these machines have a really interesting topology!
4-way SMT (each hyperthread as a full AVX-512 unit), 64 GiB of DRAM, and 16GiB of on-die HBM.
The HBM & DRAM layout is configurable too, so the HBM could be treated as a "fast" cache or as very slow memory.

We have varied across these two axes on our four machines to deliver (hopefully) all possible configurations:
  * phi0 - No HT (64 cores), HBM Far
  * phi1 - Yes HT (256 cores), HBM Far
  * phi2 - No HT, HBM Near
  * phi3 - Yes HT, HBM Near

"HBM Far" means the HBM is treated as very slow memory and DRAM is preferred by Linux.
"HBM Near" means the HBM is treated as a very fast L3-like cache.

We ask that you do not log into the phis directly, but instead "request time" on them through Slurm (either `salloc` for interactive or `sbatch` for batch jobs, see the SLURM Getting Started file in `/tank` for more information), since these machines have such odd performance characteristics.
We want to make sure your results are sensible vs. what hardware & software you are running on.
We are doing this so that people cannot accidentally step on others' toes.
We are not going to track time-used or anything.

You have full access to `/tank` from the phis, so take advantage of that when doing development vs. running workloads on them.
Below is a command to request 16 cores from phi1:
```console
karl@dubliner:~$ salloc -p phis -w v-test-phi1 -c 16
salloc: Pending job allocation 974
salloc: job 974 queued and waiting for resources
salloc: job 974 has been allocated resources
salloc: Granted job allocation 974

karl@v-test-phi1:~$ nproc
16

karl@v-test-phi1:~$
exit
salloc: Relinquishing job allocation 974
salloc: Job allocation 974 has been revoked.

karl@dubliner:~$
```

# Future Plans
In the future, we may prevent you from logging into ***SOME*** of the machines directly.
For example, the v-test-phi machines are quite slow to develop on, making them painful to use.
We might also designate jarlsberg as a Slurm-dedicated machine.
This would make those machines ideal for doing performance evaluation, since no one would ever be logged in.
