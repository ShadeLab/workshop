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
./usearch64 -otutab merged_joined/merged.fq -zotus joined_results/zotus_v1.fa -uc joined_results/ZOTU_map.uc -otutabout joined_results/ZOTU_table.txt -biomout joined_results/ZOTU_jsn.biom -notmatchedfq ZOTU_unmapped.fq
```

## OTU route

### Precluster sequences
```
./usearch64 -cluster_fast merged_joined/nosigs_uniques_combined_merged.fastq -centroids_fastq merged_joined/denoised_nosigs_uniques_combined_merged.fastq -id 0.9 -maxdiffs 5 -abskew 10 -sizein -sizeout -sort size
```

## Reference -based OTU picking (Using Silva database)
```./usearch64 -usearch_global merged_joined/denoised_nosigs_uniques_combined_merged.fastq -id 0.97 -db /mnt/research/ShadeLab/WorkingSpace/SILVA_128_QIIME_release/rep_set/rep_set_16S_only/97/97_otus_16S.fasta  -strand plus -uc joined_results/ref_seqs.uc -dbmatched joined_results/closed_reference.fasta -notmatchedfq joined_results/failed_closed.fq
```


### Mapping the closed_reference.fasta to merged.fq (pre-dereplicated sequences) and make OTU table
```
./usearch64 -usearch_global merged_joined/merged.fq -db joined_results/closed_reference.fasta -strand plus -id 0.97 -uc joined_results/OTU_map.uc -otutabout joined_results/OTU_table.txt -biomout joined_results/OTU_jsn.biom
```              

# Using QIIME to assign taxonomy to the ZOTU/OTU table
module load QIIME

### ZOTUs
assign_taxonomy.py -i joined_results/zotus_v1.fa -o joined_results/taxonomy -r ../../SILVA_128_QIIME_release/rep_set/rep_set_16S_only/97/97_otus_16S.fasta -t ../../SILVA_128_QIIME_release/taxonomy/16S_only/97/consensus_taxonomy_7_levels.txt

#biom convert -i joined_results/ZOTU_table.txt -o joined_results/ZOTU_table.biom #--table-type='OTU table'

biom add-metadata -i joined_results/ZOTU_jsn.biom -o joined_results/ZOTU_table_tax.biom --observation-metadata-fp=joined_results/taxonomy/zotus_v1_tax_assignments.txt --sc-separated=taxonomy --observation-header=OTUID,taxonomy

##### Filtering non-bacterial/archaeal reads
filter_taxa_from_otu_table.py -i joined_results/ZOTU_table_tax.biom -o joined_results/ZOTU_table_tax_filt.biom -n D_3__Streptophyta,D_3__Chlorophyta,D_4__mitochondria,Unassigned,D_4__Mitochondria
________

#### Coverting biom file to txt
biom convert -i joined_results/ZOTU_table_tax.biom -o joined_results/zotu_table.txt --header-key taxonomy -b


### OTUs
assign_taxonomy.py -i joined_results/closed_reference.fasta -o joined_results/taxonomy_otu -r ../../SILVA_128_QIIME_release/rep_set/rep_set_16S_only/97/97_otus_16S.fasta -t ../../SILVA_128_QIIME_release/taxonomy/16S_only/97/consensus_taxonomy_7_levels.txt

biom convert -i joined_results/OTU_table.txt -o joined_results/OTU_table.from_txt_json.biom --table-type="OTU table"

biom add-metadata -i joined_results/OTU_table.from_txt_json.biom -o joined_results/otu_table_tax.biom --observation-metadata-fp=joined_results/taxonomy_otu/closed_reference_tax_assignments.txt --sc-separated=taxonomy --observation-header=OTUID,taxonomy

#### Filtering non-bacterial/archaeal reads
filter_taxa_from_otu_table.py -i joined_results/otu_table_tax.biom -o joined_results/otu_table_tax_filt.biom -n D_3__Streptophyta,D_3__Chlorophyta,D_4__mitochondria,Unassigned,D_4__Mitochondria
________
#### Coverting biom file to txt
biom convert -i joined_results/otu_table_tax_filt.biom -o joined_results/otu_table_joined_.txt --header-key taxonomy -b


## Phylogenetic analysis
#### Align sequences to SILVA db using PyNAST

##### ZOTUs
align_seqs.py -i /mnt/research/ShadeLab/WorkingSpace/Stopnisek/core_microbiota/joined_results/zotus_v1.fa -o alignment -t 97_otus_aligned.fasta

##### OTUs
align_seqs.py -i /mnt/research/ShadeLab/WorkingSpace/Stopnisek/core_microbiota/joined_results/closed_reference.fasta -o joined_results/alignment -t 90_otus_aligned.fasta

filter_alignment.py -i joined_results/alignment/closed_reference_aligned.fasta -o joined_results/alignment/filtered_alignment

make_phylogeny.py -i joined_results/alignment/filtered_alignment/closed_reference_aligned_pfiltered.fasta -o joined_results/joined_set_otu.tre


