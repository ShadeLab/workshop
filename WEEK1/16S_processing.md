## Retreive sequences from the RTSF
After sequencing is done, RTSF will notify you and send you a quality report file along with password and link to retreive the samples.

You can use FileZilla or any other downloading tools or use the following command on HPCC:

First login  with `ssh <MSUNetID>@rsync.hpcc.msu.edu`
Without ssh into any node do following:
```
module load GCCcore/6.4.0 wget/1.19.4

wget -r -np -nH --ask-password ftps://shadea@titan.bch.msu.edu/20180420_16S-V4_ITS_PE
```

The following steps are based on using Illumina PE sequencing platform and USEARCH v10 for read processing.

_Some examples of workflows can be found here_: 
- [Alan 1](https://github.com/ShadeLab/PAPER_Bowsher_mSystems_2019_16sRatio_CTCstain/blob/master/Bean_Soil_HPCC_Analysis.txt), 
- [Alan 2](https://github.com/ShadeLab/PAPER_MimulusRecipTransplant_Submitted/blob/master/1%20-%20Sequence%20Analysis/HPCC_analysis.md),
- [Keara](https://github.com/ShadeLab/PAPER_GradySorensenStopnisek_NatComm_2019/blob/master/Usearch_Workflow.md),
- [Fina](https://github.com/ShadeLab/PAPER_Bintarti_2019_Apple/blob/master/16SrRNAgene_SeqWorkflow.md)

To use USEARCH you have two options:
```
1. /mnt/research/rdp/public/thirdParty/usearch10.0.240_i86linux64 (PATH to RDP)
2. /mnt/research/ShadeLab/WorkingSpace/usearch64 (copy to your working folder)  
```

Create several folders in your working directory:
```
mkdir logFiles
mkdir QC
mkdir mergedFastq
mkdir trimmed
mkdir results
mkdir raw_reads    #if you copy your reads from the Sequences/raw_sequences/
```

__Need to unzip the files before running FastQC and USEARCH!__
```
gunzip raw_reads/*.gz
```

__Remember that file names need to contain R1 and R2 string for USEARCH!__
If they don't use following command:
```
for f in ./*_1*.fastq
do 
	newname=$( echo $f | sed -r 's/_R1/_1/' )
	mv $f $newname 
done
```
Above command will replace nameID__1__L001.fastq to nameID__R1__L001.fastq. You would have to do the same for the reverse reds files (R2). 

## Check quality of the reads 
For that purpose use [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) which is available on HPCC.

```
module purge
module load FastQC/0.11.5-Java-1.8.0_162

#create a loop to process all raw and unzipped files in the folder where they exist (e.g. Seqences/raw_sequences/YOUR_FOLDER)
for file in Seqences/raw_sequences/YOUR_FOLDER/*.fastq
do
fastqc -f fastq ${file} -o QC/
done
```

## Step 1: Merging and filtering paired end reads

```
./usearch64 -fastq_mergepairs raw_reads/*R1*.fastq -relabel @ -fastq_maxdiffs 10 -fastq_minmergelen 250 -fastq_maxmergelen 300 -fastq_maxee 1 -fastqout mergedFastq/merged.fq
```

## Step 2: Trim primers 
Usually this is not necessary for the 16S rRNA amplicon sequences obtained from the RTSF because they remove the barcodes, primers and PhiX. 

```
cutadapt --discard -a ATTAGAWACCCBDGTAGTCC -a GTGCCAGCMGCCGCGGTAA -o mergedFastq/cut_merged.fq mergedFastq/merged.fq
```
If using cutadapt make sure that in the next step the input file is 'cut_merged.fq'!

## Step 3: Dereplicate (finding unique) sequences 
```
./usearch64 -fastx_uniques mergedFastq/merged.fq -fastqout mergedFastq/uniques_merged.fastq -sizeout
```
The -sizeout option specifies that size annotations should be added to the output sequence labels.

## Step 4: Remove singletons
```
./usearch64 -sortbysize mergedFastq/uniques_merged.fastq -fastqout mergedFastq/nosig_uniques_merged.fastq -minsize 2
```

## Step 6-7: Clustering and mapping
## ZOTU route
### 6.1 Predicting biological sequences = ZOTUs
```
./usearch64 -unoise3 mergedFastq/uniques_merged.fastq -zotus results/zotus.fa -tabbedout results/zotus_report.txt 
```

__Importnat!!!__
Output from the previous command is not compatible with later steps. This is because usearch requires capital letters (and numbers) for the ZOTUs to operate but it generates names as 'Zotu1'
Run following command:
```
sed -i 's/Zotu/ZOTU/g' results/zotus.fa
```

### 7.1 Mapping reads to ZOTUs
```
./usearch64 -otutab mergedFastq/merged.fq -zotus results/zotus.fa -uc results/ZOTU_map.uc -otutabout results/ZOTU_table.txt -biomout results/ZOTU_jsn.biom -notmatchedfq results/ZOTU_unmapped.fq
```

## OTU route

###  Step 6.2.1: Reference -based OTU picking (Using Silva database)
```
./usearch64 -usearch_global mergedFastq/nosigs_uniques_merged.fastq -id 0.97 -db /mnt/research/ShadeLab/WorkingSpace/SILVA_128_QIIME_release/rep_set/rep_set_16S_only/97/97_otus_16S.fasta -strand plus -uc results/ref_seqs.uc -dbmatched results/closed_reference.fasta -notmatched results/failed_closed.fa
```
###  Step 6.2.2: De novo OTU clustering 
```
./usearch64 -sortbysize results/failed_closed.fa -fastaout results/sorted_failed_closed.fasta

./usearch64 -cluster_otus results/sorted_failed_closed.fasta -minsize 2 -otus results/denovo_otus.fasta -relabel OTU_dn_ -uparseout results/denovo_otu.up --threads 40
```

Combine both results using following command
```
cat results/closed_reference.fasta results/denovo_otus.fasta > results/full_rep_set.fasta
```

### Step 7.2: Mapping reads to OTUs
```
./usearch64 -usearch_global mergedFastq/merged.fq -db results/full_rep_set.fasta -strand plus -id 0.97 -uc results/OTU_map.uc -otutabout results/OTU_table.txt -biomout results/OTU_jsn.biom
```              

## Step 8: Assign taxonomy

#### Using USEARCH
It is possible to assign taxonomy with USEARCH using the __sintax__ command, however the latest compatible version of SILVA is v123.

#### Using QIIME
Install [QIIME](http://qiime.org/install/install.html) in home directory.

##### ZOTUs
```
assign_taxonomy.py -i results/zotus_v1.fa -o results/taxonomy -r /mnt/research/ShadeLab/WorkingSpace/SILVA_128_QIIME_release/rep_set/rep_set_16S_only/97/97_otus_16S.fasta -t /mnt/research/ShadeLab/WorkingSpace/SILVA_128_QIIME_release/taxonomy/16S_only/97/consensus_taxonomy_7_levels.txt

biom convert -i results/ZOTU_table.txt -o results/ZOTU_table.biom #--table-type='OTU table'

biom add-metadata -i results/ZOTU_jsn.biom -o results/ZOTU_table_tax.biom --observation-metadata-fp=results/taxonomy/zotus_tax_assignments.txt --sc-separated=taxonomy --observation-header=OTUID,taxonomy

biom convert -i results/ZOTU_table_tax.biom -o results/zotu_table.txt --header-key taxonomy -b
```

##### OTUs
```
assign_taxonomy.py -i results/closed_reference.fasta -o results/taxonomy_otu -r /mnt/research/ShadeLab/WorkingSpace/SILVA_128_QIIME_release/rep_set/rep_set_16S_only/97/97_otus_16S.fasta -t /mnt/research/ShadeLab/WorkingSpace/SILVA_128_QIIME_release/taxonomy/16S_only/97/consensus_taxonomy_7_levels.txt

biom convert -i results/OTU_table.txt -o results/OTU_table.from_txt_json.biom --table-type="OTU table"

biom add-metadata -i joined_results/OTU_table.from_txt_json.biom -o joined_results/otu_table_tax.biom --observation-metadata-fp=joined_results/taxonomy_otu/closed_reference_tax_assignments.txt --sc-separated=taxonomy --observation-header=OTUID,taxonomy

biom convert -i results/otu_table_tax_filt.biom -o results/otu_table_joined.txt --header-key taxonomy -b
```


## Phylogenetic analysis (needed for UniFrac)

### Alignments using Muscle
First you have to download the Muscle tool (https://www.drive5.com/muscle/)
```
/muscle3.8.31_i86linux64 -in /mnt/research/ShadeLab/.../results/full_rep_set.fasta -out /mnt/research/ShadeLab/WorkingSpace/.../results/alignment_otus.fasta -maxiters 2 -diags1
```

### Constructing phylogenetic tree using FastTree
```
ml icc/2018.1.163-GCC-6.4.0-2.28  impi/2018.1.163
ml FastTree/2.1.10 

FastTree -gtr -nt results/alignment_otus.fasta > results/otuTree_fasttree.tre
```
