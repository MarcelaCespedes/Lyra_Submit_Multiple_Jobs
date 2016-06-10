# Lyra Submit Multiple Jobs
Making submitting multiple jobs to the Lyra HPC easy! 

This script is a no-dependency, easy to use solution for submiting multiple jobs to Lyra where each job requires different parameters to be loaded. Could be as something as simple as guaranteeing the random number generator seed is different for each job to a script that loads 50 parameters that change from job to job.

What originally took messing around with PBS scripts and creating multiple R scripts and pbs files then submitting `qsub` command after `qsub` command has been replaced a single .csv file and a single call of 

```shell
./pbsMulti <R_script.R> <parameter.csv>
```

# Installation
Copy the **subJobs.pbs** and **pbsMulti.sh** files into the directory that contains the R script you wish to pass arguments to. 

While using modules except `R/3.2.4_gcc` is not supported, you can add additional modules by editing the **subJobs.pbs** file to include below the module load R line, for example:

### R Jags package
``` shell
module load R/3.2.4_gcc
module load jags/4.1.0/gcc/4.4.7
```

### gdal and gdalUtils package
```shell
module load R/3.2.4_gcc
module load gdal/2.1.0/gcc/4.4.7
module load proj.4/4.9.3/gcc/4.4.7
```

## Potential Permissions Error
If you experience the following error 
```shell
-bash: ./pbsMulti.sh: Permission denied
```
then your version doesn't have execute permissions to run. This can be fixed by calling `chmod 555 pbsMulti.sh` to give read and execute permissions.

# Usage
## CSV file
The crux of this script is centred around the .csv file that holds the job specific parameters. All tables presently follow the structure:

| JOBNAME | WALLTIME | MEMORY | PARAMETER 1 | PARAMETER 2 | PARAMETER 3 |
| --- | --- | --- | --- | --- | --- |
| Example_Name | 10:30:00 | 100mb | 1 | '\"Some String\"' | 0.1 |
| Another_Example | 50:00:00 | 120gb | 0.01 | '\"Hello, World!\"' | 3.1428 |


As can be seen in the above example, the first 3 columns are restricted and must be:

1. Jobname: The name given to this sepcific job. It should act as a unique identifier for output files generated by R and the HPC.
2. Walltime: Maximum time to dedicate to this job in the format <hours>:<minutes>:<seconds> such that in the first row 10:30:00 is asking for 10hours and 30minutes while the example in the second row is asking for 50hours.
3. Maximum memory to allocate to this job. It is recommended to not ask for excessive memmory. The `<Jobname>.o<JobID>` gives both the walltime and memory statistics to aid in trimming requests for successive job submissions.

Values in any other columns are passed directly to the R script as arguments. The above example contains 3 parameters, however the script scales up and down depending on the number of columns used. 

### Restrictions
At present there are two major restrictions

1. Parameters must be numerical and strings
2. Strings can not contain commas since they conflict with the commas that seperate values in the .csv file. 

## Calling the script
Actually calling the script is as easy as using the standard `qsub` command. The template below needs two addition parameters: the R script to be called by the HPC and also where the parameters will be sent as well as the csv file to load the parameters. 
```shell
./pbsMulti <R_script.R> <parameter.csv>
```

For example, to call the [Rand Matrix example](https://github.com/A-Simmons/Lyra_Submit_Multiple_Jobs/tree/master/Rand_Matrix_Example), again we would need to copy the **subJobs.pbs** and **pbsMulti.sh** files into the Rand_Matrix_Example folder 
```shell
./pbsMulti rand_matrix_script.R rand_matrix_data.csv
```
# Letting Your R Script Accept Arguments
Using [`commandArgs(TRUE)`](https://stat.ethz.ch/R-manual/R-devel/library/base/html/commandArgs.html) from **base-R** we can pull all the arguments sent from the job submission script to our R script into a character vector with:
```R
args<-commandArgs(TRUE)
```
Now it's just a matter of pulling all the arguments out by parsing the text stored in each element of `args` and evaluating the parsed text. As an example of pulling out the first three arguments
```R
someVar1 <- eval(parse(text=args[1]))
someVar2 <- eval(parse(text=args[2]))
someVar3 <- eval(parse(text=args[3]))
```
If we don't know if three arguments are going to be sent we can use default values and use `length(args)` to check how many arguments were actually sent
```R
someVar1 <- someDefault1
someVar2 <- someDefault2
someVar3 <- someDefault3
nargs <- length(args)
```
And then use nargs to override the defaults
```R
if (nargs >= 1) {
  someVar1 <- eval( parse(text=args[1]))
  if (nargs >= 2) {
    someVar2 <- eval( parse(text=args[2]))
    if (nargs >= 3) {
      someVar3 <- eval( parse(text=args[3]))
    }
  }
}
```
or similarly
```R
if (nargs >= 1) {
  someVar1 <- eval( parse(text=args[1]))
}
if (nargs >= 2) {
  someVar2 <- eval( parse(text=args[2]))
}
if (nargs >= 3) {
  someVar3 <- eval( parse(text=args[3]))
}
```
Once your arguments/parameters have been loaded you're free to use R like you normally would. In our super basic Rand Matrix Example we use three parameters to define a matrix of different dimensions filled with random numbers. 
```R
args<-commandArgs(TRUE)

# Set some defaults
seed <- 1
n <- 10
m <- 10

# Replace defaults with arguments if they exist
nargs = length(args)
if (nargs >= 1) {
  seed <- eval( parse(text=args[1]))
  if (nargs >= 2) {
    n <- eval( parse(text=args[2]))
    if (nargs >= 3) {
      m <- eval( parse(text=args[3]))
    }
  }
}
set.seed(seed)
print(c(seed, n, m))
print(matrix(rexp(200, rate=.1),nrow=n,ncol=m))
```
which produces something like for `seed = 2000`, `n = 5`, `m=5` ([RAND_TEST3](https://github.com/A-Simmons/Lyra_Submit_Multiple_Jobs/blob/master/Rand_Matrix_Example/Expected_Output/RAND_TEST3.out) from the Rand Matrix Example). Actual values may differ from computer to computer.
```R
> print(c(seed, n, m))
[1] 2000    5    5
> print(matrix(rexp(200, rate=.1),nrow=n,ncol=m))
          [,1]      [,2]     [,3]       [,4]      [,5]
[1,] 19.590618 11.969166 2.322704  1.5313663  3.509013
[2,]  4.328520  6.403742 1.476302 15.6184668 21.942411
[3,] 11.414899  2.247588 4.978604  0.8421314 30.152508
[4,] 12.574571 20.976608 2.936833  3.9904137 16.256799
[5,]  6.266144 22.492503 5.813020 23.0723690  5.369473
```

# Full Simple Example
In this example we will go through more thoroughly the Rand Matrix Example (all files found [here](https://github.com/A-Simmons/Lyra_Submit_Multiple_Jobs/tree/master/Rand_Matrix_Example)). We're going to submit a rather small amount of jubs to the HPC, 5. Each job will have different parameters and in some cases a different number of parameters. 
The csv we're working with (found [here](https://github.com/A-Simmons/Lyra_Submit_Multiple_Jobs/blob/master/Rand_Matrix_Example/rand_matrix_data.csv)) is (it should be noted the header row is not actually in the csv file. Functionality to allow headers is being worked on)

| JOBNAME | WALLTIME | MEMORY | Seed | Rows | Cols |
| --- | --- | --- | --- | --- | --- |
|RAND_TEST1|0:05:00|10mb|1| | |
|RAND_TEST2|0:05:00|10mb|1000|5| |
|RAND_TEST3|0:05:00|10mb|2000|5|5 |
|RAND_TEST4|0:05:00|10mb|3000|1|7 |
|RAND_TEST5|0:05:00|10mb|4000|6|1 |

Logged into Lyra I can check all the necessary files are in my current working directory with `ls -l`. We can see the **pbsMulti.sh**, **subJob.pbs**, **rand_matrix_data.csv** and **rand_matrix_script.R** are in our directory. 
```shell
user@lyra04:~/ShellScript_Example/Rand_Matrix_Example> ls -l
total 16
-r-xr-xr-x 1 user default 1113 Jun  8 14:40 pbsMulti.sh
-rw-r--r-- 1 user default  159 Jun  8 14:41 rand_matrix_data.csv
-rw-r--r-- 1 user default  409 Jun  8 14:41 rand_matrix_script.R
-rwxr--r-- 1 user default  201 Jun  8 14:40 subJob.pbs
```

Moving onwards I can submit the jobs to the HPC with `./pbsMulti.sh rand_matrix_script.R rand_matrix_data.csv`
```shell
user@lyra04:~/ShellScript_Example/Rand_Matrix_Example> ./pbsMulti.sh rand_matrix_script.R rand_matrix_data.csv 

### RAND_TEST1 ###
PARAM 1: 1
665655.pbs

### RAND_TEST2 ###
PARAM 1: 1000
PARAM 2: 5
665656.pbs

### RAND_TEST3 ###
PARAM 1: 2000
PARAM 2: 5
PARAM 3: 5
665657.pbs

### RAND_TEST4 ###
PARAM 1: 3000
PARAM 2: 1
PARAM 3: 7
665658.pbs

### RAND_TEST5 ###
PARAM 1: 4000
PARAM 2: 6
PARAM 3: 1
665659.pbs
```
We notice that the script outputs the parameters for each job under the head `### JOBNAME ###`. This doesn't change anything functionally and a quiet mode is being considered for implementation it does serve as a useful platform for checking for any anomalous parameters that could arise from using strings with commas or other problems. 

Looking at he outputs, visible either through a file explorer if you've mounted the hps file server or using the list directory command `ls -l` there are a heap of **.e**, **.o** and **.out** files. In fact, there is one for each job. 
```shell 
user@lyra04:~/ShellScript_Example/Rand_Matrix_Example> ls -l
total 56
-r-xr-xr-x 1 user default 1113 Jun  8 14:40 pbsMulti.sh
-rw-r--r-- 1 user default  159 Jun  8 14:53 rand_matrix_data.csv
-rw-r--r-- 1 user default  409 Jun  8 14:41 rand_matrix_script.R
-rw------- 1 user default    0 Jun  8 14:56 RAND_TEST1.e669853
-rw------- 1 user default  133 Jun  8 14:56 RAND_TEST1.o669853
-rw------- 1 user default 2517 Jun  8 14:56 RAND_TEST1.out
-rw------- 1 user default    0 Jun  8 14:56 RAND_TEST2.e669854
-rw------- 1 user default  133 Jun  8 14:56 RAND_TEST2.o669854
-rw------- 1 user default 1926 Jun  8 14:56 RAND_TEST2.out
-rw------- 1 user default    0 Jun  8 14:56 RAND_TEST3.e669855
-rw------- 1 user default  133 Jun  8 14:56 RAND_TEST3.o669855
-rw------- 1 user default 1566 Jun  8 14:56 RAND_TEST3.out
-rw------- 1 user default    0 Jun  8 14:56 RAND_TEST4.e669856
-rw------- 1 user default  133 Jun  8 14:56 RAND_TEST4.o669856
-rw------- 1 user default 1529 Jun  8 14:56 RAND_TEST4.out
-rw------- 1 user default    0 Jun  8 14:56 RAND_TEST5.e669857
-rw------- 1 user default  133 Jun  8 14:56 RAND_TEST5.o669857
-rw------- 1 user default 1507 Jun  8 14:56 RAND_TEST5.out
-rwxr--r-- 1 user default  201 Jun  8 14:40 subJob.pbs
```
The **.e** files should be empty unless an error occured. The **.o** files will be mostly empty, containing just cpu, welltime and memory statistics of the job. The **.out** file contains the results we're interested in for this example. Anything printed to the console in R is saved to this file; along with the standard R intro spiel [RAND_TEST1.out](https://github.com/A-Simmons/Lyra_Submit_Multiple_Jobs/blob/master/Rand_Matrix_Example/Expected_Output/RAND_TEST1.out) contains the outputs from our simple code to construct a random matrix. 

```R
> args<-commandArgs(TRUE)
> 
> # Set some defaults
> seed <- 1
> n <- 10
> m <- 10
> 
> # Replace defaults with arguments if they exist
> nargs = length(args)
> if (nargs >= 1) {
+   seed <- eval( parse(text=args[1]))
+   if (nargs >= 2) {
+     n <- eval( parse(text=args[2]))
+     if (nargs >= 3) {
+       m <- eval( parse(text=args[3]))
+     }
+   }
+ }
> set.seed(seed)
> 
> 
> print(c(seed, n, m))
[1]  1 10 10
> print(matrix(rexp(200, rate=.1),nrow=n,ncol=m))
           [,1]      [,2]       [,3]       [,4]      [,5]      [,6]       [,7]
 [1,]  7.551818 13.907351 23.6451525 14.3528534 10.798811  4.222424  0.8967408
 [2,] 11.816428  7.620299  6.4189259  0.3726853 10.282469 21.787726 11.0817666
 [3,]  1.457067 12.376036  2.9412039  3.2401015 12.922616 32.177890  2.4726425
 [4,]  1.397953 44.239342  5.6586552 13.2046793 12.531054  5.578294 15.7198685
 [5,]  4.360686 10.545432  1.0607262  2.0351035  5.546414  5.946177 48.3281274
 [6,] 28.949685 10.352439  0.5943916 10.2272588  3.012830  9.773958  4.3113213
 [7,] 12.295621 18.760352  5.7871246  3.0174093 12.931247  2.098666 27.3038931
 [8,]  5.396828  6.547466 39.5893285  7.2521430  9.945558  3.094479 11.3683142
 [9,]  9.565675  3.369335 11.7331211  7.5154269  5.141743 11.059363  8.1336825
[10,]  1.470460  5.884797  9.9681296  2.3502745 20.078324  7.741878  8.3700649
            [,8]       [,9]     [,10]
 [1,] 17.8476540  3.8619356 10.350971
 [2,] 23.1247163 10.0825646  4.474519
 [3,] 29.0988727  8.1851419 10.436085
 [4,]  2.8559098  0.5926121  2.608282
 [5,]  3.8878677 22.8385347  6.812291
 [6,]  0.5205545  8.0417091  2.637383
 [7,]  3.5187050 15.8369608  4.466057
 [8,] 15.6524135 12.3379151  2.106069
 [9,]  8.1453581 13.4564402  1.325714
[10,] 27.5924379 21.0037723  3.488883
> 
> proc.time()
   user  system elapsed 
  0.192   0.012   0.209 
  ```
  Notice that in the CSV table we didn't specify the number or rows or cols; instead it defaulted to 10 for each in the R script. Let's have a look at [RAND_TEST4.out](https://github.com/A-Simmons/Lyra_Submit_Multiple_Jobs/blob/master/Rand_Matrix_Example/Expected_Output/RAND_TEST4.out) where we did define the number of rows and cols, 1 and 7 resepctively.
  ```R
  > print(c(seed, n, m))
[1] 3000    1    7
> print(matrix(rexp(200, rate=.1),nrow=n,ncol=m))
         [,1]      [,2]     [,3]     [,4]     [,5]     [,6]     [,7]
[1,] 2.582148 0.9671692 5.588815 9.444745 13.65639 7.254105 23.30046
```
Hopefully this example has highlighted how the parameters from the csv file is passed into the R script and interacted with. 

# Task Lists
- [x] Bash script completely automated. User only needs to edit their .csv file and Rscript for basic needs 
- [ ] Add functionality to load more modules than just R
- [ ] Allow headers in CSV
- [ ] Add a quiet mode