# Metagenomics
## Processing and analysis of shotgun sequencing data

### Useful resorces:

- https://astrobiomike.github.io/genomics/metagen_anvio#anvio-time

- https://github.com/mblstamps/stamps2019/wiki

- https://github.com/metagenome-atlas/atlas


## How to retrieve samples?

#### [JGI IMG](https://img.jgi.doe.gov/)

![jgi image](jgi.png)

#### [MG-RAST](https://www.mg-rast.org/)
![mgrast image](mgrast.png)

#### [NCBI SRA db](https://www.ncbi.nlm.nih.gov/sra)
![ncbisra image](ncbisra.png)

#### Your own
![precious image](precious.png)


##### Using SRAdb

Download [SRA toolkit](https://trace.ncbi.nlm.nih.gov/Traces/sra/sra.cgi?view=software) is required.

Example of job script to download SRA samples:
```
#!/bin/bash --login
########## Define Resources Needed with SBATCH Lines ##########
 
#SBATCH --time=1:00:00             # limit of wall clock time - how long the job will run (same as -t)
#SBATCH --ntasks=1                  # number of tasks - how many tasks (nodes) that you require (same as -n)
#SBATCH --cpus-per-task=1           # number of CPUs (or cores) per task (same as -c)
#SBATCH --mem=16G                    # memory required per node - amount of memory (in bytes)
#SBATCH --job-name SRR_download      # you can give your job a name for easier identification (same as -J)
#SBATCH -A shadeash-colej
#SBATCH --mail-user=stopnise@msu.edu
 
########## Command Lines to Run ##########

cd /mnt/research/ShadeLab/WorkingSpace/Stopnisek/plant_core/                  
 
sed 's/^[ \t]*//;s/[ \t]*$//' < srr_ids.txt   # remove any trailing whitespaces

mkdir /mnt/scratch/stopnise/cabbage_PRJNA437232/

for sample in $(<srr_ids.txt)
do
	
	sratoolkit.2.9.2-centos_linux64/bin/fastq-dump -I --skip-technical --dumpbase --clip --split-files ${sample} --outdir /mnt/scratch/stopnise/cabbage_PRJNA437232/
done
 
scontrol show job $SLURM_JOB_ID     ### write job information to output file
```

Check also John Quensen's tutorial for downloading samples from SRA ([LINK](http://john-quensen.com/tutorials/downloading-sequences-from-ncbis-sra/)).

## Trimming and read QC


## Assembly
Many assembly tools are available, each with some pros and cons Overall   


#### [Megahit](https://github.com/voutcn/megahit)

```
module spider MEGAHIT/1.2.4

megahit --12 <file_name> -o <output_dir> --k-list 21,39,59,79,99,141,255 -t 64
```
#### [SPAdes](https://github.com/ablab/spades)

SPAdes takes as input paired-end reads, mate-pairs and single (unpaired) reads in FASTA and FASTQ. 

```
ml GCC/5.4.0-2.26  OpenMPI/1.10.3
module load SPAdes/3.13.0

spades.py --meta --12 <file_name> -o <output_dir> -t 64 -k 21,33,55,77,99,127
```
<output_dir>/scaffolds.fasta contains resulting scaffolds (recommended for use as resulting sequences)


#### [IDBA-UD](https://github.com/loneknightpy/idba)
Requires use of FASTA files. 
__IMPORTANT__: IDBA assemblers are designed for short reads (around 100bp). If you want to assemble paired-end reads with longer read length, please modify the constant kMaxShortSequence in src/sequence/short_sequence.h to support longer read length.

```
ml GCC/6.4.0-2.28
ml IDBAUD/1.1.0

# can be also installed through conda: conda install -c bioconda idba

```

#### MASURCA

### Investigating assembly statistics

[MetaQuast](http://quast.sourceforge.net/metaquast.html) is commonly used tool to assess the assembly quality.
You can download it throuugh conda ('conda install -c bioconda quast') or just find it on HPCC ('module spider quast'). 
