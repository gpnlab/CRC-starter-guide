# **CRC Starter Guide**

## **CRC Basics**
If something is not talked about in this document please refer to https://crc.pitt.edu/ for additional information. If information is not available, you can submit a help ticket for more specialized help. 

Please use this resource for any sort of study data processing. Running pipelines on subjects 1 at a time can be slow, but with the cluster we can run multiple subjects all at once. However, **do not upload any protected health information.** Data that is already on CEREBRO is ok to upload.

Please fill out this form after performing any data processing: [link to form](https://forms.office.com/Pages/ResponsePage.aspx?id=ifT5nqDg606HzDpSYRL9Dex2ALApgKpOobBotlehtyxUMTBZRU1VSUlHWjNSNTJHQVY5UVFFTlRCNS4u). Please use the form to reduce duplicate processing, track processed datasets, and credit those who contributed to the data processing.

Make sure to acknowledge the use of the Pitt CRC in abstracts and papers https://crc.pitt.edu/about/acknowledge
### High Performance Computing Terminology
- Clusters
    - H2P Cluster
    - HTC Cluster 
    - Node
        - SMP
            - Shared Memory Processing
        - MPI
            - Message Passing Interface
        - HTC
            - High Throughput Computing
        - GPU
            - Graphics Processing Unit
        - Cores/Node
            - effectively, number of things that can be done at the same time 
        - Mem/node and mem/core
            - Limits the amount of data that can actively be used in processing
            - Similar to RAM
        - scratch
            - Limits the amount of data that can be quickly accessed for computing 
            - similar to disk space 
            - don't worry about this
- SLURM
    - Job Multiplexing
    - see below
- Service Unit (SU)
    - see below
- Containers
    - Singularity
    - Docker
    
### Service units (SU)
- Credits that are consumed when running jobs on the crc 
- Usually 1 hour of runtime is equivalent to 1 SU 
    - This not the case for GPUs 
    - [Check here](https://crc.pitt.edu/user-support/faculty-allocations-and-user-accounts/service-units) for more details regarding the usage of GPU SUs
- use the `crc-usage` command to check the total number of SUs still available

### Connecting to the CRC
- You can use ssh to connect to either the HTC or H2P clusters for the crc
    - `username@htc.crc.pitt.edu`
    - `username@h2p.crc.pitt.edu`
- if you are not on the pitt wifi network you will need to connect to the pitt VPN to access the cluster 
    - [Click here for installation instructions](https://www.technology.pitt.edu/services/pittnet-vpn-globalprotect)
    - To download the linux version go to the pitt software download center from [my.pitt.edu](my.pitt.edu)
- You can also setup a VS Code interactive session with [instructions from this link](https://crc.pitt.edu/user-manual/slurm-workload-manager/cluster-interactiveremote-computing-vs-code)
- You can connect within your browser [using this jupyterhub link](https://crc.pitt.edu/Access-CRC-Web-Portals)

### Data Paths
- You login into the `/home/haizenstein/username` folder 
    - this is limited to 75 GB of storage shared between everyone 
- For all study data please use `/ix1/haizenstein/`
    - This path has up to 5 TB of storage
- Both paths can be accessed from all compute nodes on the cluster 

## **SLURM**
Workload manager used to allocate resources and startup multiple jobs

### Common SLURM Commands 
- sbatch
    - individual jobs
    - Multiple jobs (arrays)
    - GPU
    - Interactive
- crc-squeue (squeue)
- crc-sinfo (sinfo)

### sbatch Parameters
- sbatch allows you to set parameters to indicate what resources your job will require and will allocate it onto the cluster.
    - cluster/partition parameters are used to identify the specific node required. 
    - time/qos are used to let the workload allocator know how long you'll need a node and can impact how long a job will be queued for
- [Check here](https://help.rc.ufl.edu/doc/Sample_SLURM_Scripts) for examples of job submission scripts. Make sure to double check the submission script before submitting

### Slurm script files
- sbatch parameters and commands are able to be called from slurm scripts. These files are usually use the `.slurm` file extension.
- Use `#SBATCH` to indicate the declaration of an sbatch parameter
- Everything else within the script is assumed to be the run using the program indicated in the shebang (`#!`)
    - e.g. `#!/bin/bash` for bash 
    - Always run `crc-job-stats` at the end of your jobs

### Environment Variables
- within each slurm job there are some useful environment variables that can be used
- `$SLURM_ARRAY_TASK_ID`for array jobs lets you multiplex processing for multiple jobs
- See [here for more environment variables that are available](https://docs.hpc.shef.ac.uk/en/latest/referenceinfo/scheduler/SLURM/SLURM-environment-variables.html)

### Modules
- modules must be loaded at the beginning of each job before use. 
- `module load module_name` will load a module
- `module spider module_name` will search for a module
- `module --help` will print out information regarding the command

### Job Multiplexing Example

Let's say you have 100 images you want to process. Normally you would run the processing pipeline on each subject 1 at a time. Freesurfer takes about ~6 hours per subject and thus would take a total of 25 days. Using a cluster you want to run _as many as possible at the same time_. This can drastically reduce the amount of time the processing will take with the trade off of being harder to setup. How should we structure our job submission?

First, Freesurfer has significant performance increases using multiple cores (`recon-all -all -s $subject -parallel -openmp NUM_CORES`) and any more than 8 cores doesn't seem to improve performance. Using the HTC Cluster, there are 32 nodes with either 64 per node (ignore the 48 core nodes for this example). At 4 cores per subject we can fit 16 subjects per node. Freesurfer also needs about 8 GB of memory per subject for a total of 96 GB per node, fortunately these nodes all have >512GB of memory. To process all subjects, we will need a total of 400 cores which is much lower than the total number of cores available. Additionally, we will only need a little more that 6 full nodes of the 32 available nodes. 

Knowing the above we can begin to setup our job. Since we are well below the total capacity of the cluster, we can submit 1 job per subject to slurm. We can submit multiple jobs by using `#SBATCH --array=0-99` to let slurm know we want to submit 100 jobs. This sets the `$SLURM_ARRAY_TASK_ID` environment variable for each job which we can use to determine which subject should be processed. We should then create a list of subjects that we wish to process in a new file. This may contain either ID values or full folder paths either of which can be adapted to run on the CRC. For this example we will use assume full paths are provided. Assuming the file is named `subject_list.txt`, we can grab individual folder paths with the following code 

```bash
# get file name
folders=($(cat subject_list.txt))
subject_folder=${folders[${SLURM_ARRAY_TASK_ID}]}

echo "parsing subject: "$subject_folder
```  

We can also set `#SBATCH --cpus-per-task=4` to request 4 cores or cpus per job and `#SBATCH --nodes=6.25` to let slurm know the minimum number of nodes needed.  Since we have an idea of how long Freesurfer should take we should also fill add `#SBATCH --time 24:00:00` to limit the maximum time used per job.

With that we have the core requirements for submitting multiple jobs at once. To fully complete the code we will also need load in freesurfer and call recon-all as well as set paths for logs and add various other features see `/ihome/haizentein/liw82/crc-tutorial/tutorial.slurm` for a full example script

### Interactive sessions
You can use the following command to start an interactive bash shell session for 1 hour on 1 core: `srun -n1 -t02:00:00 --pty bash`

Most standard sbatch parameters can be applied to specify the specific node/cluster, extend the time, or any thing else that is needed. Changing the `--pty` parameter can change the terminal that is instantiated (e.g. `--pty zsh`)


## **GPU Usage**
- since we have limited GPU resources please do not use the cluster for large model training unless everything has been validated
- make sure to set your cluster to gpu and your partition to the required gpu partition. 
- Please use the V100 partition unless you know you will need more than 32 GB of GPU memory. 
    - For smaller models please use one of our lab computer or reach out to [l.wang@pitt.edu](mailto:l.wang@pitt.edu)
- If you would like to do a code review please reach out to [l.wang@pitt.edu](mailto:l.wang@pitt.edu)

### ML Model Training Checklist
- [ ] All random elements should be seeded and use the same seed
- [ ] Model is able to parse input data and produce the desired output datatype. The desired output should be backpropogated through the model to update the model weights.
- [ ] Data is split into at least 1 train, validation, and test set
- [ ] Training code should iterate through all training and validation dataset splits for a given number of epochs
    - [ ] Training code should only iterate through the test set(s) at the end of training 
- [ ] Model weights are saved at the end of training
    - Best performing model weights should be saved periodically
- [ ] Metrics used to evaluate model performance are saved for the training and validation set during training. Test set metrics should be saved at the end of training only.
- [ ] Hyperparameters should be easily accessable either within the code or via a configuration file


### GPU SOP
1. Check if you are using the right cluster and any special considerations for that cluster
2. Make sure your data is uploaded
3. Write your slurm file and executable for a simple test case to verify that your code will work as intended using the data on the cluster and the computational environment is setup correctly. Check specifically for the following: 
	1. All required modules are loaded
	2. All code is free of syntax errors and obvious runtime errors
	3. All data that is required is stored within `/ix/haizenstein` and is being downloaded to the computation node
    4. All code contains the required metric monitoring and logging required for analysis and benchmarking
4. Start an interactive session using a gtx 1080 node to verify that your code is working. 
    1. Check the ML Model Training Checklist to make sure your code contains all necessary components for
    1. Your code should be able to start model training and iterate through your validation set without fail.
	1. For models requiring large amounts of memory please use Pytorch lightning to verify the memory usage of your model
	2. For multi-gpu configurations please ensure that you are using [DDP](https://pytorch.org/tutorials/intermediate/ddp_tutorial.html)
    3. Make sure you are saving model checkpoints as needed and logging experiment results
5. Check slurm parameters for correct cluster resource requests
6. Submit job 
