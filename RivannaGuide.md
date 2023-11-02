# Rivanna Guide

Author: Teague R Henry

11/2/2023

## Before you log in:
-	You will need to be added to the correct MyGroups by your PI (me).
-	Download and install the Cisco VPN Client, and log into UVA More Secure Network

## Logging in: 
-	For PC users, install MobaXTerm
-	For Mac users, install XQuartz from xquartz.org

Once installed, open up either MobaXterm or your Xterm (if on Mac. Make sure you are using Xterm rather than the regular terminal).

- For MobaXterm you can log in via ssh into rivanna.hpc.virginia.edu
- For Mac, run `ssh -Y <YOURNETBADGE>@rivanna.hpc.virginia.edu` substituting your NetBadge ID in.

If you are successful, you will see an enter your password prompt. When you type, it will look like nothing is happening, this is to be expected as Linux systems don't show passwords being typed.

## Once Logged In:

Once you are logged in, you will see text and a prompt that likely says something like `bash$<somenumbers>` If so, you have successfully logged into the system.

The most important tools for navigating around the system are the following commands:
- `pwd` this stands for present working directory, and prints out where you are.
- `cd` this stands for change directory, and allows you to change where you are. To use this, you can use absolute pathing (i.e. `cd /home/ycp6wm` would take you directly to my home directory). You can also use relative pathing, using `.` for the current working directory, and `..` for one directory above. For example, if you were sitting in the `/home/` main  directory, you could navigate to my home directory via `cd ./ycp6wm`. If you were in my home directory, you could return to the main directory via `cd ..`
- `ls` this function lists the files in your current working directory. I tend to use it with additional options `ls -lah` which prints out more information, prints out human readable file sizes, and shows hidden files. 
- `mkdir` this function makes a new directory, and can use the same type of pathing as `cd`. For example, if I wanted to make a directory in my home directory, and I was currently sitting in my home directory, I would type `mkdir ./anewdirectory`
- `rm` this function deletes files. It can be very dangerous to use. Just be very careful with it.
- `cat <somefilename>` prints the contents of a file to the terminal. You cannot modify the file, but this lets you see what's in it.

Every user on Rivanna has a home directory, which allows for approximately 20gb of storage. This tends to be sufficient for simulation studies, so you will primarily be working out of your home directory. You will not have permissions to access any other directories (with few exceptions).

## Modules in Rivanna: 

Software in Rivanna is installed into modules which users have to activate. Once activated, you will be able to use that software. 

- to list all modules available, you can use `module avail`
- to get help for a given module, use `module spider <modulename>`
- to activate a module, you use `module load <modulename>/<versionnumber>`. For example if you wanted to activate R, you would use `module load R/4.2.1`
- If you are always going to want to have access to a module, you need to use the `module save` command. This saves the current module setup to your default configuration. If you don't do this, you will have to reload all your modules every time you log in.

## Activating R:

R takes a little bit more work to activate, as it has prerequisites. To activate R/4.2.1 (most current version as of 11/2/2023) run the following commands.

```
module load gcc/9.2.0 openmpi/3.1.6
module load R/4.2.1
module save
```

Once you've run those commands, you can start R by just typing `R`. This will bring up the R console prompt. To quit, just type `q()` (I'd suggest never saving your environment, it can cause issues down the line.)

You will want to install your R packages within R. To do this, use the `install.packages` function. Once you install your packages, you shouldn't need to install them again for your Rivanna account. The packages are installed into your home directory in a hidden folder.

## Rivanna as a Cluster

Rivanna is a cluster, which means it's most useful when running many separate jobs (such as many different conditions in a simulation study.) Rivanna uses SLURM as its job management system. The idea behind this system is that you submit jobs to the cluster, SLURM then decides when to run these jobs (depending on priority, current load, and job settings), and once the jobs are complete, you work with the results. Importantly, these jobs are not interactive (they can be, but that's very advanced). 

We will be primarily submitting jobs via an R package, more on that later. But we will be looking at job progress on the main command line. The following commands are very useful

- `squeue -u <yournetbadge>` This prints out a list of all currently running jobs you've submitted. If you don't use the `-u` option, you are going to see all jobs running not just your own. If you have no jobs running, your list will be empty.

- `sacct -u <yournetbadge> --starttime $(date -d "7 days ago" +"%Y-%m-%d") --format=JobID,JobName,State,Reason,Elapsed,TotalCPU,Start,End` This more complicated function gives quite a lot of information for all jobs you submitted in the last 7 days by you. You would use this command to see which jobs completed and which jobs failed, along with the reason why the job might have failed. This is quite useful when diagnosing what went wrong with a batch submission. Note: I used ChatGPT to write this command, it is quite useful in forming complex commands in SLURM.

- `scancel <jobid>` This cancels a currently running job by job ID. You can use wildcards here, which is useful for batch submitted jobs (i.e. to cancel all jobs that start with 1234567, you would use `scancel 1234567*`)

## Submitting jobs via Rslurm 

The `rslurm` package is a very useful R package for automatically submitting slurm jobs. I won't get into the specific details of the functions, as there are examples of them in the example simulation study. I will detail a number of finicky bits when writing R code for rslurm submission.

- Each job that is launched by `rslurm` is run in a different subdirectory. This means that if you `source` in additional files, you need to be careful about file pathing. I like to use absolute paths. For example, if I wanted to source a file in `blah` directory in my home directory, I would need to put in the `run_wrapper` function `source("/sfs/qumulo/qhome/ycp6wm/blah/afiletosource.R")` Importantly, `/sfs/qumulo/qhome/` is the true filepath to the home directories (rather than `/home`, which is an alias). So, you want to use `/sfs/qumulo/qhome/` when you filepath to any files you want to source.
- I'll emphasize that each job is run as a separate R instance, which means you either have to load in all your packages per job (by including the `library` commands in the `run_wrapper` function), or use some of the advanced options in the `rslurm` submission functions.
- The output of each job is saved as a .RDS file, which are a pain to work with separately. `rslurm` has a function to gather the output together, but that requires the job object which is produced when you run the `rslurm` commands to submit jobs. Make sure you always save this file (I usually call it `sjob`) using the `save()` R function.
- The printed log of each job is contained in the job running folder as a .Rout file. These are plaintext files that contain what you would see in the console if you had run this interactively. You can look at these files to troubleshoot.

Now, you might have reached the stage where you've uploaded your Simulation code as a single R file (let's call it `sim_run.R`), which correctly sources any helper function files (which you've also uploaded to the appropriate folders in Rivanna.) How do you now launch all the simulations? There are two ways:

- The lazy way (which is what I do all the time): Copy and paste the code from sim_run.R into the terminal after you've started R. This works, it works just fine, but it can have issues.
- The correct way (which I should do much more): Run the `sim_run.R` from the terminal using `R CMD BATCH sim_run.R` This runs the R script file, which then launches all the jobs. After I run this command, I check to make sure that the jobs are queued by using the `squeue -u <yournetbadge>` command. IMPORTANTLY: If the job list is blank, you likely have an error in the sim_run.R file. If the jobs are pending, then they immediately disappear, they either have completed quickly or they have failed due to errors. 

Putting this all together, the steps to launching a simulation study are:
1. Make a new directory for your simulation study in your home directory.
2. Upload (via MobaXterm or the ondemand system) your code into that directory.
3. Navigate to that directory using `cd`
4. Run your main simulation file using `R CMD BATCH`
5. Check to make sure the jobs are queued using `squeue`. If there are issues, look at the outfiles.


## Other Useful Rivanna Things:

### Editing text files directly on Rivanna

You can use a text editor to modify files directly on Rivanna. This is extremely useful. I use a program called `vim`. `vim` is a very, _very_, complicated program that, in theory, allows you to customize your workflow to a very fine level. I don't use it like that, I just use it to modify text files. Here is how you use it.

- To start `vim` and open up a file, you just use `vim thefilename.ext`. So, if you wanted to modify the `sim_run.R` file, all you need to do is run `vim sim_run.R`. 
- When `vim` starts up, it is not in edit mode. To get into edit mode, type `a`. You will see text at the bottom of the screen saying `INSERT`. You can now type and modify the file. To leave insert mode, hit the `esc` key.
- To save and quit out of `vim`, you need to leave insert mode, and type `:wq`, and hit enter. This stands for `write, then quit`, which saves the file (overwriting the original file).

### Configuring your Rivanna experience

Linux systems can be configured to your preference in a very fine grained way. One example of this is with _aliases_. Aliases are ways of specifying a complex command or filepath in a more simple way. For example, you could have an alias for `squeue -u <yournetbadge>` as just `squeue`, so you don't need to constantly be writing `-u <yournetbadge>.` If you have a long file path that you always navigate to, you can alias it, for example `/a/long/file/path/to/some/directory/you/want/to/go/to` could be `target_dir`, and you could navigate to it by using `cd target_dir`. 

To add aliases, you need to modify your `.bashrc` file (note the . at the beginning of the name, this is a hidden file in your home directory). Open up the file using `vim .bashrc`, and add at the end of the file your aliases. The correct structure of the alias command is `alias <whatyouwantittobe>='<thelongcommand>'`. So, for example, I have an alias for `squeue` as `alias squeue='squeue -u ycp6wm'`. Note the lack of spaces between the `=` and its surroundings.
