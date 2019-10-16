The workflow is modified from Bonito lab and [Fina's workflow](https://github.com/ShadeLab/PAPER_Bintarti_2019_Apple/blob/master/ITSgene_SeqWorkflow.md)


## Step 1: Merge PE reads
```
./usearch64 -fastq_mergepairs /rawreads/*R1*.fastq -reverse /rawreads/*R2*.fastq -fastq_maxdiffs 10 -fastq_minmergelen 50 -relabel @ -fastqout mergedFastq/merged.fastq
```

There are few options in USEARCH that allow you to inspect the quality and quality stats of processed reads
```
./usearch64 -fastq_eestats2 mergedFastq/merged.fq -output stats_eestats2_USEARCH_joined.txt -length_cutoffs 100,500,1
./usearch64 -fastx_info mergedFastq/merged.fq -secs 5 -output stats_fastxinfo_USEARCH_joined.txt
```

## Step 2: Remove primers
For amplification of ITS we use EMP recommended primer set.

```
cutadapt -g CTTGGTCATTTAGAGGAAGTAA -a GCATCGATGAAGAACGCAGC -f fastq -n 2 --discard-untrimmed --match-read-wildcards -o mergedFastq/cut_merged.fastq mergedFastq/merged.fastq > logFolder/cut_adpt_results.txt
```

## Step 3: Filter reads
```
./usearch64 -fastq_filter cmergedFastq/ut_merged.fastq -fastq_maxee 1 -fastq_trunclen 200 -fastq_maxns 0 -fastaout mergedFastq//filtered_cut_merged.fa -fastqout mergedFastq/filtered_cut_merged.fastq
```

## Step 4: Dereplicate sequences
```
./usearch64 -fastx_uniques mergedFastq/filtered_cut_merged.fastq -fastaout mergedFastq/uniques_filtered_cut_merged.fasta -sizeout
```

## Step 5: Remove singletons
```
./usearch64 -sortbysize mergedFastq/uniques_filtered_cut_merged.fasta -fastaout mergedFastq/nosig_uniques_filtered_cut_merged.fasta -minsize 2
```

## Step 6: Clustering

### Step 6.1: Reference-based clustering

```
./usearch64 -usearch_global mergedFastq/nosig_uniques_filtered_cut_merged.fasta -id 0.97 -db /mnt/research/ShadeLab/UNITE_v7.2/sh_refs_qiime_ver7_97_s_01.12.2017.fasta -strand plus -uc results/ref_seqs.uc -dbmatched results/UNITE_reference.fasta -notmatched results/UNITE_failed_closed.fq
```

### Step 6.2: Sorting by size and and de-novo clustering

```
./usearch64 -sortbysize results/UNITE_failed_closed.fq -fastaout results/sorted_UNITE_failed_closed.fa

./usearch64 -cluster_otus results/sorted_UNITE_failed_closed.fa -minsize 2 -otus results/DENOVO_otus.fasta -uparseout results/uparse_otus.txt -relabel OTU_dn_
```

Combine the rep sets 

```
cat results/UNITE_reference.fasta results/DENOVO_otus.fasta > results/full_rep_set.fna
```

## Step 7: Mapping

```
./usearch64 -usearch_global mergedFastq/merged.fastq -db results/full_rep_set.fna -strand plus -id 0.97 -uc results/OTU_map.uc -otutabout results/OTU_table_ITS.txt
-biomout OTU_jsn_ITS.biom
```

##  Step 8: Assign taxonomy using CONSTAX

Please refer to how "Running CONSTAX on the MSU HPCC on lab guru : https://my.labguru.com/knowledge/documents/330

Few things to know:
1. __YOU HAVE TO RUN CONSTAX WITHIN THE OLD CENTOS 6 HPCC SYSTEM!__
2. copy results/full_rep_set.fna into /home/<user>/CONSTAX_hpcc/otus/
3. change the PATH in: mnt/home/<user>/CONSTAX_hpcc/config
4. output directories: outputs, taxonomy_assignments, training_files
5. output file 'home/<user>/CONSTAX_hpcc/outputs/consensus_taxonomy.txt' is the one you use 


