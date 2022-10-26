# donau-slurm-wrappers

#### 介绍
donau-slurm-wrappers提供了一些slurm命名风格的脚本，方便习惯使用slurm的用户快速上手donau的业务。
目前支持的命令有: 

   作业提交命令: srun, sbatch

   作业查询命令: squeue, sacct
   
   作业控制命令: scancel, scontrol
   
   节点查询命令: sinfo

#### 软件架构
python2/python3

#### 依赖
1. 集群已安装Donau Scheduler, 安装版本 >= HPC22.0.0 B015, 安装节点可正常提交Donau-Cli命令。
2. 脚本依赖python模块dateutil.relativedelte, 执行pip install python-dateutil安装。

#### 安装教程
1. 从网址 https://gitee.com/openeuler/donau-slurm-wrappers 下载压缩包, 解压至用户的家目录
2. 更改脚本的属主为DONAU的CLI用户, 修改权限为555， 例如当前用户为ccs_cli，则执行   
     
     $chown -R ccs_cli:ccs_cli donau-slurm-wrappers-master && chmod -R 555 donau-slurm-wrappers-master   

3. 配置CLI用户的环境变量PATH, 将"donau-slurm-wrappers-master/cmd/"的绝对路径添加进PATH。（建议直接修改
  用户的配置文件 ~/.bashrc 并source）
  
    

#### 使用说明

1.srun命令  
    srun用于提交阻塞式作业。如果直接使用srun提交作业，会默认转成dsub –Kco提交阻塞式作业，如果srun
 指定--pty，则转换成dsub -I提交交互式作业。对于交互式作业，需要提交到指定队列执行。用户需要参考HPC产品文档，
 配置支持交互式作业的队列。用户通过sinfo查看当前partition，然后使用srun --pty –p指定partition提交交互式作业。
    
    
    
    $srun --usage
    Usage: srun [-n ntasks] [-o out] 
            [--open-mode={append|truncate}] [-e err]
            [-c ncpus] [-p partition] [-t minutes]
            [-D path] [-J jobname][--gid=group]
            [--dependency=type:jobid][--comment=name]
            [--nodelist=hosts][--exclude=hosts]
            [--prolog=fname] [--epilog=fname]
            [--gpus-per-task=n]
            executable [args...]
    
举例:
    
    srun -b "17:18 11/09/22" -c 2 -n 1 -J "job_name" -t 200 user_command

转换为donau命令
    
    dsub –Kco -N 1 -S "2022/11/09 17:18:00" --name "job_name" -T "3h20m" -R "cpu=2"  user_command
    
2.sbatch命令  
    sbatch用于提交脚本作业。最终会把命令转换为dsub -s , 建议把执行的脚本放到共享目录。如果是mpi作业，
需要用--mpi指定mpi类型（目前支持的类型为openmpi, mpich, hmpi, intelmpi), 然后在脚本中的用户命令中增加环境变量
$CCS_MPI_OPTIONS。
    
    
    
    $sbatch --usage
    Usage: sbatch [-N nnodes] [-n ntasks] 
                [-c ncpus] [-p partition] [-t minutes]
                [-D path] [--mpi mpi_type] [--output file]
                [--open-mode={append|truncate}] [--error file] 
                [--chdir=directory] [-J jobname] [--gid=group] 
                [--dependency=type:jobid] [--comment=name] 
                [--ntasks-per-node=n] [--nodelist=hosts]
                [--exclude=hosts][--export[=names]] [--gpus-per-task=n]
                executable [args...]
                
普通作业举例:
   
    #脚本文件  
    $cat hello.sh
    #!/bin/sh
    #SBATCH --comment test_job
    #SBATCH -J test_job
    #SBATCH -n 2
    #SBATCH -o /tmp/log.txt
    #SBATCH -p root.default
    #SBATCH --open-mode truncate
    sleep 2
    
   执行
    
    $sbatch -b "17:18 11/09/22" -c 2 -n 4 -J "job_name" -t 200 -p q3 hello.sh
    

MPI作业举例:
    
    #脚本文件    
    $cat mpi_job.sh 
    #! /bin/sh
    #SBATCH -o /tmp/%j.txt         #重定向输出
    #SBATCH --comment=script_job   #指定comments
    #SBATCH -n4                    #指定task个数
    mpirun $CCS_MPI_OPTIONS user_application
    
   执行
    
    $sbatch --mpi openmpi mpi_job.sh
           
                
3.squeue命令   
    squeue支持查询未完成(RUNNING, PENDING, SUSPENDED)的作业信息，对应的DONAU作业的状态为
RUNNING, PENDING/WAITING, STOPPED。显示结果中"NODELIST(REASON)"的未调度原因，直接使用DONAU
中task的"MAIN_STATE_REASON_CODE", 具体原因参考《HPC产品文档》。


    
    $squeue --usage
    Usage: squeue [--job jobid] [-n name] [-p partitions]
              [-t states] [-u user_name] [--usage] [-l]   

举例:


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
    

4.scancel命令
    scancel支持终止未完成的作业，对应的DONAU命令为djob -T, 执行scancel后的作业状态为CANCELLED。  


    
     $scancel --usage  
     Usage: scancel [-n job_name] [-p partitions]
              [-t PENDING | RUNNING | SUSPENDED] [--usage] [-u user_name] [job_id]  
    
    
5.sinfo命令  
    sinfo用于查询当前集群的节点和分区信息(SLURM的PARTITION和DONAU的QUEUE是两种概念，为了用户对指定QUEUE
进行操作，将QUEUE和PARTITION对应)。
    
    
    
    $sinfo --usage
    Usage: sinfo [-lN] [-p partition] [-n nodes]
    
举例:
    
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
    
6.scontrol命令  
    scontrol命令用于停止/恢复作业，对应DONAU的djob -S和djob -R。
    
    
    
    $scontrol --usage
    scontrol [<OPTION>] [<COMMAND>]  
        Valid <OPTION> values are:                                             
        --help     show this help message                                             

        Valid <COMMAND> values are:
        resume <jobid_list>      resume previously suspended job
        suspend <jobid_list>     suspend specified job  


7.sacct命令  
    sacct查询作业的记账信息，支持查询的作业状态有COMPLETED, FAILED, CANCELLED, RUNNING，对应的DONAU
作业状态为SUCCEEDED, FAILED, RUNNING)。

    
   
    $sacct --usage
    Usage: sacct [--name] [-S starttime] [-E endtime] [-r partition]
              [-s state] [-u user] [-j jobs] [-b]
              
举例:
    
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
    
#### 注意事项

一. donau-slurm-wrappers是对donau的一些cli命令的二次封装，命令的执行依赖于cli本身的规则。脚本不支持root用户
调用, 同时在实际使用中可能会遇到token校验失败的问题。常见的token失败场景如下：

1). 报错: "error: access token does not exist, please execute command dconfig to get token"  
    说明: 用户在执行CLI命令前，没有使用dconfig命令获取身份验证的token，需要执行dconfig命令获取token。
    或者用户在执行dconfig命令后，由于某种原因导致存储token的文件丢失，这种情况需要再次执行dconfig命令。

2). 报错: "error: token expired! refresh token does not exist, please execute command dconfig to get token"  
    说明: 这种情况是因为token过期，且之前dconfig命令时没有返回用于刷新的refresh token。

3). 报错: "token unauthorized"  
    说明: 用户未授权，没有加入用户组或管理员组。
    
    更多的错误及解决方案参考《HPC产品文档》。

二. 为方便调试，可通过设置环境SLURM_TO_DONAU_DEBUG开启日志打印，开启后每个脚本会在/tmp目录下生成
对应的日志文件，文件名为脚本名.UID.PID。该脚本暂不支持自动删除，需要用户手动删除。
    
#### 参与贡献

1.  Fork 本仓库
2.  新建 Feat_xxx 分支
3.  提交代码
4.  新建 Pull Request


#### 特技

1.  使用 Readme\_XXX.md 来支持不同的语言，例如 Readme\_en.md, Readme\_zh.md
2.  Gitee 官方博客 [blog.gitee.com](https://blog.gitee.com)
3.  你可以 [https://gitee.com/explore](https://gitee.com/explore) 这个地址来了解 Gitee 上的优秀开源项目
4.  [GVP](https://gitee.com/gvp) 全称是 Gitee 最有价值开源项目，是综合评定出的优秀开源项目
5.  Gitee 官方提供的使用手册 [https://gitee.com/help](https://gitee.com/help)
6.  Gitee 封面人物是一档用来展示 Gitee 会员风采的栏目 [https://gitee.com/gitee-stars/](https://gitee.com/gitee-stars/)
