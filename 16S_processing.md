# Retreive sequences from the RTSF
After sequencing is done, RTSF will notify you and send you a quality report file along with password and link to retreive the samples.

You can use FileZilla or any other downloading tool or use the following command on HPCC:
```
wget -r -np -nH --ask- password ftps://shadea@titan.bch.msu.edu/20180420_16S-V4_ITS_PE
```

The following steps are based on using Illumina PE sequencing platform and USEARCH for read processing.

To use USEARCH you have two options:
```
1. /mnt/research/rdp/public/thirdParty/usearch10.0.240_i86linux64 (PATH to RDP)
2. /mnt/research/ShadeLab/WorkingSpace/usearch64 (copy to your working folder)  
```

Creat several folders in your working foledr:
```
mkdir logFiles
mkdir QC
mkdir mergedFastq
mkdir trimmed
mkdir results
```

## Step 1: Merging and filtering paired end reads

```
./usearch64 -fastq_mergepairs /raw_reads/*R1*.fastq -relabel @ -fastq_maxdiffs 10 -fastq_minmergelen 250 -fastq_maxmergelen 300 -fastq_maxee 1 -fastqout /mergedFastq/merged.fq
```

## Step 2: Trim primers 
Usually this is not necessary for the 16S rRNA amplicon sequences obtained from the RTSF because they remove the barcodes, primers and PhiX. 

```
cutadapt --discard -a ATTAGAWACCCBDGTAGTCC -a GTGCCAGCMGCCGCGGTAA -o /mergedFastq/cut_merged.fq /mergedFastq/merged.fq
```

## Step 3: Dereplicate (finding unique) sequences 
```
./usearch64 -fastx_uniques /mergedFastq/merged.fq -fastqout mergedFastq/uniques_merged.fastq -sizeout
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
sed -i 's/Zotu/ZOTU/g' results/zotus_v1.fa
```

### 7.1 Mapping reads to ZOTUs
```
./usearch64 -otutab mergedFastq/merged.fq -zotus results/zotus.fa -uc results/ZOTU_map.uc -otutabout results/ZOTU_table.txt -biomout results/ZOTU_jsn.biom -notmatchedfq ZOTU_unmapped.fq
```

## OTU route

### Step 6.2.1: Precluster sequences
```
./usearch64 -cluster_fast mergedFastq/nosigs_uniques_merged.fastq -centroids_fastq mergedFastq/denoised_nosigs_uniques_merged.fastq -id 0.9 -maxdiffs 5 -abskew 10 -sizein -sizeout -sort size
```

###  Step 6.2.2: Reference -based OTU picking (Using Silva database)
```./usearch64 -usearch_global mergedFastq/denoised_nosigs_uniques_merged.fastq -id 0.97 -db /mnt/research/ShadeLab/WorkingSpace/SILVA_128_QIIME_release/rep_set/rep_set_16S_only/97/97_otus_16S.fasta  -strand plus -uc results/ref_seqs.uc -dbmatched results/closed_reference.fasta -notmatchedfq results/failed_closed.fq
```

### Step 7.2: Mapping the closed_reference.fasta to merged.fq (pre-dereplicated sequences) and make OTU table
```
./usearch64 -usearch_global mergedFastq/merged.fq -db results/closed_reference.fasta -strand plus -id 0.97 -uc results/OTU_map.uc -otutabout results/OTU_table.txt -biomout results/OTU_jsn.biom
```              

## Using QIIME to assign taxonomy to the ZOTU/OTU table

### ZOTUs
```
assign_taxonomy.py -i results/zotus_v1.fa -o results/taxonomy -r /mnt/research/ShadeLab/WorkingSpace/SILVA_128_QIIME_release/rep_set/rep_set_16S_only/97/97_otus_16S.fasta -t /mnt/research/ShadeLab/WorkingSpace/SILVA_128_QIIME_release/taxonomy/16S_only/97/consensus_taxonomy_7_levels.txt

biom convert -i results/ZOTU_table.txt -o results/ZOTU_table.biom #--table-type='OTU table'

biom add-metadata -i results/ZOTU_jsn.biom -o results/ZOTU_table_tax.biom --observation-metadata-fp=results/taxonomy/zotus_tax_assignments.txt --sc-separated=taxonomy --observation-header=OTUID,taxonomy

biom convert -i results/ZOTU_table_tax.biom -o results/zotu_table.txt --header-key taxonomy -b
```

### OTUs
```
assign_taxonomy.py -i results/closed_reference.fasta -o results/taxonomy_otu -r /mnt/research/ShadeLab/WorkingSpace/SILVA_128_QIIME_release/rep_set/rep_set_16S_only/97/97_otus_16S.fasta -t /mnt/research/ShadeLab/WorkingSpace/SILVA_128_QIIME_release/taxonomy/16S_only/97/consensus_taxonomy_7_levels.txt

biom convert -i results/OTU_table.txt -o results/OTU_table.from_txt_json.biom --table-type="OTU table"

biom add-metadata -i joined_results/OTU_table.from_txt_json.biom -o joined_results/otu_table_tax.biom --observation-metadata-fp=joined_results/taxonomy_otu/closed_reference_tax_assignments.txt --sc-separated=taxonomy --observation-header=OTUID,taxonomy

biom convert -i results/otu_table_tax_filt.biom -o results/otu_table_joined.txt --header-key taxonomy -b
```


## Phylogenetic analysis
### Align sequences to SILVA db using PyNAST

__ZOTUs__
```
align_seqs.py -i /mnt/research/ShadeLab/WorkingSpace/Stopnisek/core_microbiota/results/zotus.fa -o alignment_zotu -t 97_otus_aligned.fasta
```

__OTUs__
```
align_seqs.py -i /mnt/research/ShadeLab/WorkingSpace/Stopnisek/core_microbiota/results/closed_reference.fasta -o alignment_otu -t 97_otus_aligned.fasta
```

filter_alignment.py -i alignment_zotu/closed_reference_aligned.fasta -o joined_results/alignment/filtered_alignment

make_phylogeny.py -i joined_results/alignment/filtered_alignment/closed_reference_aligned_pfiltered.fasta -o joined_results/joined_set_otu.tre


