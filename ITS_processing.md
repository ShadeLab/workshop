
mv /mnt/research/ShadeLab/WorkingSpace/Stopnisek/core_microbiota/bulk_soil/*ITS*.fastq /mnt/research/ShadeLab/WorkingSpace/Stopnisek/core_microbiota/ITS/raw_reads/

for f in ./*_ITS*.fastq; do newname=$( echo $f | sed -r 's/cITS/_ITS/' ); mv $f $newname; done


#ITS
/mnt/research/rdp/public/thirdParty/usearch10.0.240_i86linux64 -fastq_mergepairs ../rawreads/*R1*.fastq -reverse ../rawreads/*R2*.fastq -fastq_maxdiffs 10 -fastq_minmergelen 50 -relabel @ -fastqout mergedfastq/merged.fastq

./usearch64 -fastq_mergepairs ./raw_reads/*R1*.fastq -relabel @ -fastq_minmergelen 10 -fastq_trunclen 250  -fastqout ./mergedfastq_joined/merged.fq

Totals:
   6309012  Pairs (6.3M)
   4605875  Merged (4.6M, 73.00%)
   2810840  Alignments with zero diffs (44.55%)
   1595715  Too many diffs (> 5) (25.29%)
    107422  No alignment found (1.70%)
         0  Alignment too short (< 16) (0.00%)
         0  Merged too short (< 10)
    551955  Staggered pairs (8.75%) merged & trimmed
    200.59  Mean alignment length
    263.32  Mean merged length
      1.06  Mean fwd expected errors
      0.86  Mean rev expected errors
      0.10  Mean merged expected errors

#vsearch -fastq_stats mergedfastq_joined/merged.fq -log stats_results_VSEARCH_joined.txt

#./usearch64 -fastq_eestats2 mergedfastq_joined/merged.fq -output stats_eestats2_USEARCH_joined.txt -length_cutoffs 100,500,1

#./usearch64 -fastx_info mergedfastq_joined/merged.fq -secs 5 -output stats_fastxinfo_USEARCH_joined.txt

# Dereplicate sequences
./usearch64 -fastx_uniques ./mergedfastq_joined/merged.fq -fastqout mergedfastq_joined/uniques_combined_merged.fastq -sizeout

00:34 3.1Gb   100.0% DF
00:34 3.1Gb  4605875 seqs, 753502 uniques, 524833 singletons (69.7%)
00:34 3.1Gb  Min size 1, median 1, max 194207, avg 6.11
00:50 3.1Gb   100.0% Writing mergedfastq_joined/uniques_combined_merged.fastq

# Remove singletons
./usearch64 -sortbysize mergedfastq_joined/uniques_combined_merged.fastq -fastqout mergedfastq_joined/nosigs_uniques_combined_merged.fastq -minsize 2

00:03 555Mb   100.0% Reading mergedfastq_joined/uniques_combined_merged.fastq
00:03 521Mb  Getting sizes                                                   
00:03 527Mb  Sorting 228669 sequences
00:03 528Mb   100.0% Writing output

# Precluster sequences
./usearch64 -cluster_fast mergedfastq_joined/nosigs_uniques_combined_merged.fastq -centroids_fastq mergedfastq_joined/denoised_nosigs_uniques_combined_merged.fastq -id 0.9 -maxdiffs 5 -abskew 10 -sizein -sizeout -sort size

00:01 171Mb   100.0% DF
00:01 176Mb  228669 seqs (tot.size 4081042), 228669 uniques, 0 singletons (0.0%)
00:01 176Mb  Min size 2, median 2, max 194207, avg 17.85
00:01 190Mb   100.0% DB
00:01 193Mb  Sort size... done.
00:26 229Mb   100.0% 11123 clusters, max size 337193, avg 366.9
00:26 229Mb     0.0% Writing centroids to mergedfastq_joined/denoised_nosigs_uniques_combined_merge
00:26 229Mb   100.0% Writing centroids to mergedfastq_joined/denoised_nosigs_uniques_combined_merged.fastq
                                                                                                          
      Seqs  228669 (228.7k)
  Clusters  11123 (11.1k)
  Max size  337193 (337.2k)
  Avg size  366.9
  Min size  2
Singletons  0, 0.0% of seqs, 0.0% of clusters
   Max mem  229Mb
      Time  26.0s
Throughput  8795.0 seqs/sec.

# Reference -based OTU picking
./usearch64 -usearch_global mergedfastq_joined/denoised_nosigs_uniques_combined_merged.fastq -id 0.97 -db /mnt/research/ShadeLab/WorkingSpace/Stopnisek/core_microbiota/ITS/UNITE_db/sh_refs_qiime_ver7_97_s_01.12.2017.fasta -strand plus -uc mergedfastq_joined/ref_seqs.uc -dbmatched mergedfastq_joined/closed_reference.fasta -notmatchedfa mergedfastq_joined/failed_closed.fq

00:00 37Mb      0.1% Reading /mnt/research/ShadeLab/WorkingSpace/Stopnisek/core_microbiota/ITS/UNITE_d
00:00 73Mb    100.0% Reading /mnt/research/ShadeLab/WorkingSpace/Stopnisek/core_microbiota/ITS/UNITE_db/sh_refs_qiime_ver7_97_s_01.12.2017.fasta
00:00 40Mb      0.1% Masking (fastnucleo)                                                             
00:01 40Mb    100.0% Masking (fastnucleo) 
00:02 41Mb    100.0% Word stats          
00:02 41Mb    100.0% Alloc rows
00:03 140Mb   100.0% Build index
00:03 173Mb  CPU has 20 cores, defaulting to 10 threads

WARNING: Max OMP threads 1

00:26 176Mb   100.0% Searching denoised_nosigs_uniques_combined_merged.fastq, 24.2% matched

# Sorting and de-novo clustering
./usearch64 -sortbysize mergedfastq_joined/failed_closed.fq -fastaout mergedfastq_joined/sorted_failed_closed.fq
./usearch64 -cluster_otus mergedfastq_joined/sorted_failed_closed.fq -minsize 2 -otus mergedfastq_joined/denovo_otus.fasta -relabel OTU_dn_ -uparseout denovo_out.up

01:22 71Mb    100.0% 2834 OTUs, 1198 chimeras

#removing OTUs with length less than 250bp
./usearch64 -sortbylength mergedfastq_joined/denovo_otus.fasta -fastaout mergedfastq_joined/denovo_otus_filtered.fasta -minseqlength 250

#Combine the rep sets 
cat mergedfastq_joined/closed_reference.fasta mergedfastq_joined/denovo_otus_filtered.fasta > mergedfastq_joined/full_rep_set.fna

./usearch64 -usearch_global mergedfastq_joined/merged.fq -db mergedfastq_joined/full_rep_set.fna -strand plus -id 0.97 -uc OTU_map_joined_.uc -otutabout OTU_table_joined.txt -biomout OTU_jsn_joined.biom
00:00 44Mb    100.0% Reading mergedfastq_joined/full_rep_set.fna
00:00 10Mb    100.0% Masking (fastnucleo)                       
00:00 11Mb    100.0% Word stats          
00:00 11Mb    100.0% Alloc rows
00:00 15Mb    100.0% Build index
00:00 48Mb   CPU has 20 cores, defaulting to 10 threads

WARNING: Max OMP threads 1

20:24 52Mb    100.0% Searching merged.fq, 86.5% matched
3986302 / 4605875 mapped to OTUs (86.5%)               
20:24 52Mb   Writing OTU_table_joined.txt
20:24 52Mb   Writing OTU_table_joined.txt ...done.
20:24 52Mb   Writing OTU_jsn_joined.biom
20:24 52Mb   Writing OTU_jsn_joined.biom ...done.

# Using QIIME to assign taxonomy
assign_taxonomy.py -i mergedfastq_joined/full_rep_set.fna -o taxonomy -r /mnt/research/ShadeLab/WorkingSpace/Stopnisek/core_microbiota/ITS/UNITE_db/sh_refs_qiime_ver7_97_s_01.12.2017.fasta -t /mnt/research/ShadeLab/WorkingSpace/Stopnisek/core_microbiota/ITS/UNITE_db/sh_taxonomy_qiime_ver7_97_s_01.12.2017.txt --uclust_similarity=0.5

# Add taxonomy to OTU table
biom convert -i OTU_table_joined.txt --table-type='OTU table' -o otu_jsn_joined_filtered.biom
biom add-metadata -i otu_jsn_joined_filtered.biom -o otu_table_tax_joined_filtered.biom --observation-metadata-fp=taxonomy/full_rep_set_tax_assignments.txt --sc-separated=taxonomy --observation-header=OTUID,taxonomy

# Filter Protists and Unassigned taxa
filter_taxa_from_otu_table.py -i otu_table_tax_joined_filtered.biom -o otu_table_tax_joined_filtered.biom -n k__Protista,Unassigned
biom summarize-table -i otu_table_tax_joined_filtered.biom -o otu_joined_table_filtered_sum.txt

# Make OTU tree
./usearch64 -cluster_agg mergedfastq_joined/full_rep_set.fna -treeout mergedfastq_joined/seqs_joined.tree 


biom convert -i otu_table_tax_joined_filtered.biom -o OTUtable_ITS.txt --header-key taxonomy -b

# Rarefactin of the otu tble by the read counts (n=32416)
single_rarefaction.py -i otu_table_tax_joined_filtered.biom -o otu_table_even_ITS.biom -d 22005
biom convert -i otu_table_even_ITS.biom -o OTUtable_ITS_rarified.txt --header-key taxonomy -b

./usearch64 -beta_div OTUtable_ITS_rarified.txt -tree otus.tree -filename_prefix beta/
