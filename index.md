# Manual on Using Microsoft Azure

Azure is a cloud computing solution that provides a range of services including virtual machines, servers for data processing, computing clusters for batch jobs.

The University is pushing for Azure, so cloud computing services will be part of the same infrastructure.

A simulation that took 6 days on the lab machine took about 12 hours on the Azure Virtual Machine (VM).

Our instance is called **[vm-bric-spot-comp-modelling]**.

* Ubuntu 20.04.3 LTS (GNU/Linux 5.8.0-1043-azure x86_64)
* 3x AMD EPYC 7452 32-Core Processor
* 96 vCPUs
* 385 GB RAM
* 200 GB Storage (Shared)
* R version 4.1.2 (2021-11-01)

## Using Azure CLI to manage your VM

## Logging in

There are various solutions, like PuTTy, xRDP (seems the best for regular psychologists), ssh. I am using simple ssh.

```bash
ssh Researcher@10.115.20.7
```

You can copy files to the instances by using scp.

```bash
scp my-files-to-copy Researcher@10.115.20.7:/home/Documents/my-project/
```

## Running Your Script

There are multiple ways to run your code. You can use screen and other solutions, but a bash script is the best, because you can actually shut down the instance when finished - even when the simulations are halted due to an error. You don't want to keep paying for an idle instance.

The bash script will be a file ending in `.sh` and beginning with `#!/bin/sh` on the first line. See the example below:

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
Rscript my-unashamedly-parallelized.R | tee -a "log.$(date +"%m-%d-%y").out"

# or if you use a python script that is not executable as a program
python my-anasamedly-gpu-dependent.py | tee -a "log.$(date +"%m-%d-%y").out"

echo "End of job at"
date

# shut down instance so you don't have to pay for surplus time
# this is optional
sudo shutdown now
```

Note that I would recommend testing a dumbed-down version of your code and make sure it runs and saves what you need to disk. If the script encounters an error, it still executes the remaining code and shuts down the instance.

## Executing your script on the Virtual Machine

After having your bash script, run the following command:

```bash
# make it executable
chmod u+x my-bash-script.sh
# run command in background
nohup "./my-bash-script.sh" > output.log.out &
```

I would generally recommend to use `screen`, so that you can return to your workspace. It will make your life easier, especially if you are monitoring what you are doing or your code prints out info about the simulation. You don't have to worry about it with xRDP.

## Writing files to disk

There is a shared `/DATA` folder that can be mounted on both the instance and your local machine.

Paul is yet to explain the details to me.

## Checking jobs

If you want to see how your script is running, you could use `htop` to check on the number of cores used or check on the `output.log.out` to see where things are.

## How to load packages in your script

In most scenarios, use as few packages as possible. I would also recommend not to load metapackages like `tidyverse`. In case you want to safeguard against missing packages, here are some scripts below to install missing packages before the rest of your code is run.

```r
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

I didn't test the python version on the VM, but this seems to be at least a locally working solution:

```python
try:
    import scipy
except ImportError:
    from pip._internal import main as pip
    pip(['install', '--user', 'scipy'])
    import scipy
```