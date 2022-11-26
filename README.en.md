# donau-slurm-wrappers

#### Description
Donau-slurm-wrappers provide some scripts for Slurm users to get start with Donau Scheduler quickly.
These scripts with slurm command syntax and options execute Donau-Cli commands in the background.

Commands include:

    Job Submission Commands: srun, sbatch
    Job Query Commands     : squeue, sacct
    Job Control Commands   : scancel, scontrol
    Node Query Commands    : sinfo

#### Software Architecture
python2/python3

#### Dependence
1. Donau Scheduler has been installed, versions "HPC22.0.0 B015" or later are supported. 
The Donau-Cli command can be submitted on the installation node successfully.  
2.  Scripts depend on the python module "dateutil.relativedelta". Run the command 
"pip install python-dateutil" to install it.

#### Installation
1. Download the package from "https://gitee.com/openeuler/donau-slurm-wrappers" and decompress it to the home directory of the user.
2. Change the owner of the script to the Donau CLI user and change the permission to 555. 
For example, if the current user is ccs_cli, run the following command:
 
    $chown -R ccs_cli:ccs_cli donau-slurm-wrappers-master && chmod -R 555 donau-slurm-wrappers-master   

3. Configure the environment variable PATH of the Cli user and add the absolute path of "donau-slurm-wrappers-master/cmd/" to PATH.
(You are advised to modify file "~/.bashrc" and source it.)

#### Instructions

1.srun  
    The srun is used to submit a blocking job. Job will be regarded as a Donau block job which submitted by 
"dsub -Kco". If option "--pty" is specified, job will be regarded as Donau interactive job submitted by
"dsub -I". Users must configure queues to support interactive jobs refer to "HPC Product Documentation". Then
users can execute command "srun --pty -p queuename user_command" to submit an interactive job.


    $srun --usage
    Usage: srun [-n ntasks] [-o out][-b "HH:MM MM/DD/YY"]
            [--open-mode={append|truncate}] [-e err]
            [-c ncpus] [-p partition] [--priority=value][--pty]
            [-t minutes] [-D path][-J jobname][--gid=group]
            [--dependency=type:jobid][--comment=name]
            [--nodelist=hosts][--exclude=hosts][--export=env_vars]
            [--prolog=fname] [--epilog=fname][--exclusive]
            [--gpus-per-task=n]
            executable [args...]

INPUT ENVIRONMENT VARIABLES(lower priority than options):  

    SRUN_CPUS_PER_TASK   same as -c
    SLURM_DEPENDENCY     same as --dependency
    SLURM_WORKING_DIR    same as -D
    SLURM_EXPORT_ENV     same as --export
    SLURM_STDERRMODE     same as -e
    SLURM_EPILOG         same as --epilog
    SLURM_JOB_NAME       same as -J
    SLURM_NTASKS         same as -n
    SLURM_STDOUTMODE     same as -i
    SLURM_OPEN_MODE      same as --open-mode
    SLURM_PARTITION      same as -p
    SLURM_PROLOG         same as --prolog
    SLURM_TIMELIMIT      same as -t
    SLURM_GPUS_PER_TASK  same as --gpus-per-task

Example:
    
    srun -b "17:18 11/09/22" -c 2 -n 1 -J "job_name" -t 200 user_command
    
converts to Donau-command

    dsub –Kco -N 1 -S "2022/11/09 17:18:00" --name "job_name" -T "3h20m" -R "cpu=2"  user_command

2.sbatch  
    The sbatch is used to submit a batch script. The command will be converted to "dsub -s". You are advised
to place the script in the shared directory. Users should specify MPI type by option "--mpi" (additional option
to specify mpi type to run Donau mpi job) and add environment variables "$CCS_MPI_OPTIONS" to user-command in
the script to submit a mpi job. Supported mpi type includes openmpi, mpich, intelmpi and hmpi. 

    
    $sbatch --usage
    Usage: sbatch [-N nnodes] [-n ntasks] [-b "HH:MM MM/DD/YY"]
            [-c ncpus] [-p partition] [-e err] [--time=minutes]
            [-D path] [--mpi mpi_type] [--output file]
            [--open-mode={append|truncate}] [--error file] 
            [--priority=value] [--chdir=directory] [-J jobname] 
            [--gid=group] [--dependency=type:jobid] [--comment=name]
            [--ntasks-per-node=n] [--nodelist=hosts][--exclude=hosts] 
            [--export[=names]] [--exclusive] [--gpus-per-task=n]
            executable [args...]
                
INPUT ENVIRONMENT VARIABLES(lower priority than options): 

    SBATCH_EXPORT          same as --export
    SBATCH_ERROR           same as -e
    SBATCH_JOB_NAME        same as -J
    SBATCH_OUTPUT          same as -o
    SBATCH_OPEN_MODE       same as --open-mode
    SBATCH_PARTITION       same as -p
    SBATCH_TIMELIMIT       same as -t
    SBATCH_GPUS_PER_TASK   same as --gpus-per-task

Example of normal job:
   
    # user script
    $cat hello.sh
    #!/bin/sh
    #SBATCH --comment test_job
    #SBATCH -J test_job
    #SBATCH -n 2
    #SBATCH -o /tmp/log.txt
    #SBATCH -p root.default 
    #SBATCH --open-mode truncate
    sleep 2
    
   Execute:

    $sbatch -b "17:18 11/09/22" -c 2 -n 4 -J "job_name" -t 200 -p q3 hello.sh
    

Example of openmpi job:
    
    # user script    
    $cat mpi_job.sh 
    #! /bin/sh
    #SBATCH -o /tmp/%j.txt         # redirect output file
    #SBATCH --comment=script_job   # comments
    #SBATCH -n4                    # task number
    mpirun $CCS_MPI_OPTIONS user_application
    
   Execute:
   
    $sbatch --mpi openmpi mpi_job.sh
    
3.squeue  
    The squeue is used to query information about jobs that are not completed. Supported states include
RUNNING, PENDING and SUSPENDED, the corresponding Donau job states are RUNNING, WAITING/PENDING and STOPPED.
The value of NODELIST(REASON) in output is replaced by "MAIN_STATE_REASON_CODE" of Donau task. 
For details, see the "HPC Product Documentation".


    $squeue --usage
    Usage: squeue [--job jobid] [-n name] [-p partitions]
              [-t states] [-u user_name] [--usage] [-l]
              
INPUT ENVIRONMENT VARIABLES(lower priority than options): 

    SQUEUE_NAMES          same as -n
    SQUEUE_STATES         same as -t
    SQUEUE_USERS          same as -u
    SQUEUE_PARTITION      same as -p

Example:

    $squeue
    JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)       
      520   root.q1  default  test_st PD       0:00      - 20615
      521   root.q1  default  test_st PD       0:00      - 10101
      522   root.q1  default  test_st  S   11:28:42      1 kwephicprd18119
    
    $squeue -l
    Wed Oct 19 09:30:34 2022
        JOBID PARTITION     NAME     USER    STATE       TIME TIME_LIMI   NODES NODELIST(REASON)
          520   root.q1  default  test_st  PENDING       0:00 UNLIMITED       -            20615 
          521   root.q1  default  test_st  PENDING       0:00 UNLIMITED       -            10101
          522   root.q1  default  test_st SUSPENDE   11:29:22 UNLIMITED       1  kwephicprd18119

4.scancel  
    The scancel is used to terminate unfinished jobs. The corresponding Donau command is "djob -T." 
The job state would be CANCELLED terminated by scancel.
    
    
    $scancel --usage  
    Usage: scancel [-n job_name] [-p partitions]
              [-t PENDING | RUNNING | SUSPENDED] [--usage] [-u user_name] [job_id]
               
INPUT ENVIRONMENT VARIABLES(lower priority than options):  

    SCANCEL_NAME          same as -n
    SCANCEL_STATE         same as -t
    SCANCEL_USER          same as -u
    SCANCEL_PARTITION     same as -p  

5.sinfo  
    The sinfo is used to query the node and partition information of the current cluster. (Slurm's partition
does not correspond to Donau's queue. It provides convenient way for users to use Donau's queue by partition.)  


    $sinfo --usage
    Usage: sinfo [-lN] [-p partition] [-n nodes]
    
Example:
    
    $sinfo
    PARTITION AVAIL TIMELIMIT NODES STATE NODELIST       
    root      up    infinite  1     idle  kwephispra23857
    root.q1   up    infinite  1     idle  kwephispra23857
    root.q2   up    infinite  1     idle  kwephispra23857
    root.q3   up    infinite  1     idle  kwephispra23857
    root      up    infinite  1     mix   kwephicprd18119
    root.q1   up    infinite  1     mix   kwephicprd18119
    root.q2   up    infinite  1     mix   kwephicprd18119
    root.q3   up    infinite  1     mix   kwephicprd18119
    
    $sinfo -N
    NODELIST        NODES PARTITION STATE
    kwephispra23857 1     all       idle 
    kwephicprd18119 1     all       mix
    
6.scontrol  
    The scontrol command is used to stop/resume jobs, corresponding to "djob -S/djob -R" in Donau.
    
    
    $scontrol --usage
    scontrol [<OPTION>] [<COMMAND>]  
        Valid <OPTION> values are:                                             
        --help     show this help message                                             

        Valid <COMMAND> values are:
        resume <jobid_list>      resume previously suspended job
        suspend <jobid_list>     suspend specified job  
    
7.sacct  
    The sacct is used to display accounting data for jobs. Supported job states include COMPLETED, FAILED, 
CANCELLED, RUNNING corresponding to SUCCEEDED, FAILED, RUNNING in Donau.

    
    $sacct --usage
    Usage: sacct [--name] [-S starttime] [-E endtime] [-r partition]
              [-s state] [-u user] [-j jobs] [-b]
              
Example:

    $sacct
    JobID           JobName  Partition    Account  AllocCPUS      State ExitCode
    ------------ ---------- ---------- ---------- ---------- ---------- --------
    315             default    root.q1                     0  CANCELLED      0:9
    343             default root.defau                     1     FAILED      1:0
    519             default    root.q1                     1    RUNNING      0:0
    524             default    root.q1                     1  COMPLETED      0:0
    
    $sacct -b
    JobID             State ExitCode
    ------------ ---------- --------
    315           CANCELLED      0:9
    343              FAILED      1:0
    519             RUNNING      0:0
    524           COMPLETED      0:0
    
    
#### Precautions

1.Donau-slurm-wrappers are wrappers of some Donau CLI commands. Wrappers execution depends on the rules of 
the Donau CLI. Scripts do not support the root user and token may be verified failed in actual case. 
Common token errors are as follows:

(1). Error: "error: access token does not exist, please execute command dconfig to get token"   
     Description: Before running the CLI command, you need to run the dconfig command to obtain the token.
     Alternatively, after you run the dconfig command, the token file is lost. In this case, 
     you need to run the dconfig command again.
     
(2). Error: "error: token expired! refresh token does not exist, please execute command dconfig to get token"  
     Description: This is because the token expires and the refresh token is not returned when the dconfig 
     command is executed.
     
 3). Error: "token unauthorized"  
     Description: The user is not authorized to join the user group or administrator group.
     
     For details about more errors and solutions, see the "HPC Product Documentation".

2.Users can set the environment "SLURM_TO_DONAU_DEBUG" true to enable debug log. If enabled, each script
will generate a log file named "script_name.uid.pid" in directory "/tmp". Log files can not be deleted automatically
and need to be deleted manually. 
     
#### Contribution

1.  Fork the repository
2.  Create Feat_xxx branch
3.  Commit your code
4.  Create Pull Request


#### Gitee Feature

1.  You can use Readme\_XXX.md to support different languages, such as Readme\_en.md, Readme\_zh.md
2.  Gitee blog [blog.gitee.com](https://blog.gitee.com)
3.  Explore open source project [https://gitee.com/explore](https://gitee.com/explore)
4.  The most valuable open source project [GVP](https://gitee.com/gvp)
5.  The manual of Gitee [https://gitee.com/help](https://gitee.com/help)
6.  The most popular members  [https://gitee.com/gitee-stars/](https://gitee.com/gitee-stars/)
