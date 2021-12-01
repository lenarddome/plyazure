---
title: Running Simulations on Azure
---

Lab head: Andy Wills

**Developmental Version**

Azure is a cloud computing solution that provides a range of services
including virtual machines (VM), servers for data processing, computing
clusters for batch jobs. Azure is now on of the top 10 supercomputers in
the world ([source](https://www.top500.org/system/180024/)).

In the lab, we are using Azure to run computationally demanding jobs
that would require a supercomputer to finish in a reasonably humane
amount of time. The University of Plymouth also has a
[supercomputer](https://www.plymouth.ac.uk/about-us/university-structure/faculties/science-engineering/hpc),
but it is less supported than Microsoft Azure at BRIC and maintained by
other departments.

Set up Microsoft Azure on Linux
===============================

In the lab, we run [Linux](https://en.wikipedia.org/wiki/Linux), so this
manual is specifically aimed at getting you ready to use Azure via the
**Linux CLI** (e.g. bash, zsh, fish, ...). Most of the solutions will
also work on Windows, but they are not tested on Windows. Alternatively,
you can install Ubuntu on Windows as a subsystem, see more info
[here](https://www.microsoft.com/en-gb/p/ubuntu/9nblggh4msv6?activetab=pivot:overviewtab).

This document also assumes that you have **the right access** to Azure.
If you still need to arrange access, you won\'t be able to follow this
manual.

You should have also received the following information:

-   **username** your username for the VM.
-   **host** an IP address of the VM.
-   **resource-group** the name of your resource group.
-   **vm-name** the name of the virtual machine.

Our currently available VM instances at BRIC
--------------------------------------------

**\[vm-bric-spot-comp-modelling\]**

-   Ubuntu 20.04.3 LTS (GNU/Linux 5.8.0-1043-azure x86\_64)
-   3x AMD EPYC 7452 32-Core Processor
-   96 vCPUs
-   385 GB RAM
-   200 GB Storage (Shared)
-   R version 4.1.2 (2021-11-01)
-   Spot Pricing

Connecting to the University\'s Network
---------------------------------------

In order to connect and manage your VM instance, you will need to be
connected to `eudoram` on campus, or to be connected to the
University\'s VPN. There is a good deal of documentation on [how to set
up VPN on Linux](https://www.andywills.info/assets/pdf/vpn-setup.pdf).

Using Azure CLI to manage your VM
---------------------------------

Install Azure CLI for Linux according to your preferred method. There is
official documentation on how to do it
[here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt).

After installing it, you need to log in to your account with

```bash
az login
```

This command will open a browser window, where you will need to log in
with your University email and password. In the future, there will be
multiple and different type of instances - e.g. one with high number of
CPU that is optimized for computing and one with a high-end GPU for Deep
Convolutional Networks. You can list all VMs that are accessible to you
with

```bash
az vm list
```

before selecting which one to start for the job.

To start the VM, use

```bash
az vm start -g resource-group -n vm-name
```

To shut down the VM, you will need to use `az vm deallocate`. Simply
using `az vm stop` will stop the VM, but a stopped VM will still incur
charges.

```bash
az vm deallocate -g resource-group -n vm-name
```

Logging in
----------

There are various solutions, like PuTTy, xRDP (seems the best for
regular psychologists), ssh. I am using simple ssh.

```bash
ssh username@host
```

Enter your password when prompted.

You can copy files to the instances by using scp.

```bash
# copy file
scp my-files-to-copy username@host:/home/path/to/my-project/
# copy all things in current directory
scp *  username@host:/home/path/to/my-project/
```

The code that you should run on Azure
=====================================

When you should run your code on Azure:

-   Your script takes more then a week to run on the [local lab
    machine](https://www.andywills.info/willslab-ply/).
-   You need a lot of memory (\> 64 GB).
-   You need a lot of GPU MEMORY (\> 5 GB).
-   The simulation script can run by itself and write the results to
    disk without any human intervention.

How to install R packages
-------------------------

The root directories on the VM will not be writeable by either you or R.
The `.libPaths` command will enable you to set new target directories
for the packages you are about to install.

``r
.libPaths("home/your/new/path")
```

In most scenarios, use as few packages as possible. It is recommended
not load or install metapackages like `tidyverse`. It is guaranteed that
it will fail to install due to some obscure missing dependency. Instead,
it is better to simply use `dplyr`, `reshape2`, as these packages
contain most of the utilities used in tidyverse, such as `%>%`,
`summarize`, `mutate`. If you feel more comfortable with R, `data.table`
is the best choice, see more on the package
[here](https://github.com/Rdatatable/data.table).

How to load packages in your script
-----------------------------------

In case you want to safeguard against missing packages, here are some
scripts below to install missing packages before the rest of your code
is run.

``r
## If a package is installed, it will be loaded. If any
## are not, the missing package(s) will be installed
## from CRAN and then loaded.
## https://vbaliga.github.io/verify-that-r-packages-are-installed-and-loaded/

## First specify the packages of interest
packages = c("catlearn", "data.table",
             "psp", "doParallel")

## Now load or install&load all
package.check <- lapply(
  packages,
  FUN = function(x) {
    if (!require(x, character.only = TRUE)) {
      install.packages(x, dependencies = TRUE, ask = FALSE)
      library(x, character.only = TRUE)
    }
  }
)
```

I didn't test the python version on the VM, but this seems to be at
least a locally working solution:

```python
try:
    import scipy
except ImportError:
    from pip._internal import main as pip
    pip(['install', '--user', 'scipy'])
    import scipy
```

Note that if you are using Python, you will set the path with `--user`.

Writing files to disk
---------------------

There is a shared `/DATA` folder that can be mounted on the instance and
accessed through a personal machine. You should have received
information about where it is. I recommend you double check on the VM
and use the absolute path when saving the simulation results.

This is a 200 GB storage, but it can be increased if needed.

Executing Your Script
=====================

There are multiple ways to run your code - e.g. use screen, but a bash
script is the best. It allows you to simply shut down the instance when
finished - even when the simulations are halted due to an error. You
don't want to keep paying for an idle instance.

The bash script will be a file ending in `.sh` and beginning with
`#!/bin/sh`. See the example below:

```bash
#!/bin/sh

# allow errors so the bash script keeps running even
# if your code resulted in an error
set +e

echo "LANG" $LANG

# run script

echo "Start of job at"
date

# run script and save command line outputs to log
Rscript my-unashamedly-parallelized-simulation.R | tee -a "log.$(date +"%m-%d-%y").out"

# or if you use a python script that is not executable as a program
python3 my-embarrassingly-cpu-heavy-job.py | tee -a "log.$(date +"%m-%d-%y").out"

echo "End of job at"
date

# gracefully shut down instance so you don't have to pay for surplus time
az vm deallocate -g resource-group -n vm-name
```

Note that I would recommend testing a dumbed-down version of your code
and make sure it runs and saves what you need to disk. If the script
encounters an error, it still executes the remaining code and shuts down
the instance.

Executing your script on the Virtual Machine
--------------------------------------------

After writing, testing and uploading your bash script, run the following
command:

```bash
# make it executable
chmod u+x my-bash-script.sh
# run command in background with logging
nohup "./my-bash-script.sh" > output.log.out &
# or run command in background without logging
nohup "./my-bash-script.sh" &
```

I would generally recommend to use `screen`, so that you can return to
your workspace. It will make your life easier, especially if you are
monitoring what you are doing or your code prints out info about the
simulation. You don't have to worry about it with xRDP.

Checking jobs
-------------

If you want to see how your script is running, you could use `htop` to
check on the number of cores used or check on the `output.log.out` to
see where things are.
