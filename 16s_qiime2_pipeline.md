16s Data Qiime2 Pipeline 
-----------

This pipeline is based on qiime2 : https://docs.qiime2.org/2017.10/tutorials

Creators: microbiome group (Shixian,Sana,Lianmin)

Year: 2017

1. Qiime2 installation and data preparation
```
# installing minicoda 

wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
sh ./Miniconda3-latest-Linux-x86_64.sh

# please make sure the qiime2 is the latest version, or it would get errors in following steps

conda create -n qiime2-2017.12 --file https://data.qiime2.org/distro/core/qiime2-2017.12-conda-linux-64.txt

# activate qiime2

source activate qiime2-2017.12
```
```
# preparing the demultiplexed data (without barcode and primer), and the Metadata_16S.tsv
# moving all the R1.fastq.gz and R2.fastq.gz files in one folder (clean_data), the same as "CasavaOneEightSingleLanePerSampleDirFmt"
# rename fastq.gz files as BAQ2420.1.2_55_L001_R1_001.fastq.gz (sample identifier, the barcode sequence or a barcode identifier, the lane number, the read number, and the set number)

#!/bin/bash
for i in *.gz
do
  juged=$(echo $i | awk -F "[._]" '{print $4}')
  keep1=$(echo $i | awk -F "[._]" '{print $1}')
  keep2=$(echo $i | awk -F "[._]" '{print $3}')
  if [[ "$juged" -eq "1" ]]
  then
    mv ./$i ./$keep1.$keep2.\1_22_L001_R1_001.fastq.gz
  elif [[ "$juged" -eq "2" ]]
  then
    mv ./$i ./$keep1.$keep2.\1_22_L001_R2_001.fastq.gz
  fi 
done
```

2. Importing data
```
# generate qza file 

qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path clean_data/ \
  --source-format CasavaOneEightSingleLanePerSampleDirFmt \
  --output-path demux.qza
  
# output: demux.qza
```
```
# transfer qza to qzv which is able to be visualized in https://view.qiime2.org/, where you can also find the quality score and decide cut-off for DADA2 step

qiime demux summarize \
  --i-data demux.qza \
  --o-visualization demux.qzv
  
# output: demux.qzv
```
```
# if on mac, please use command

qiime tools view demux.qzv 
```

3. Sequence quality control and feature table construction

Trimming low quality bp ,generating feature table and representative sequences using DADA2

DADA2 is a method to infer the sample sequences in a collection of amplicon sequencing reads. DADA2 is a de novo method, and completely reference free
```
# --p-trim-left-f and --p-trim-left-r: how many bp do you want to trim from 5'end
# --p-trunc-len-f and --p-trunc-len-r: the trim position on 3'end

qiime dada2 denoise-paired \
  --i-demultiplexed-seqs demux.qza \
  --o-table table \
  --o-representative-sequences rep-seqs \
  --p-trim-left-f 0 \
  --p-trim-left-r 0 \
  --p-trunc-len-f 200 \
  --p-trunc-len-r 200
  
# output: table.qza rep-seqs.qza
```
```
# transfer table.qza and rep-seqs.qza to qzv

qiime feature-table summarize \
  --i-table table.qza \
  --o-visualization table.qzv \
  --m-sample-metadata-file Metadata_16S.tsv

qiime feature-table tabulate-seqs \
  --i-data rep-seqs.qza \
  --o-visualization rep-seqs.qzv
  
# output: table.qzv rep-seqs.qzv
```
4. Close-reference OTUs piking
```
# import reference data (Greengene)

qiime tools import \
  --type 'FeatureData[Sequence]' \
  --input-path 97_otus.fasta \
  --output-path 97_otus.qza
  
qiime tools import \
  --type 'FeatureData[Taxonomy]' \
  --source-format HeaderlessTSVTaxonomyFormat \
  --input-path 97_otu_taxonomy.txt \
  --output-path ref-taxonomy.qza
```
```
# OTUs picking
# The clustering method in vsearch is based on the publicly available information about the uclust method on the usearch website:
# http://drive5.com/usearch/manual/uclust_algo.html

qiime vsearch cluster-features-closed-reference \
  --i-table table.qza \
  --i-sequences rep-seqs.qza \
  --i-reference-sequences 97_otus.qza \
  --p-perc-identity 0.97 \
  --o-clustered-table table-cr-97.qza \
  --o-unmatched-sequences unmatched.qza \
  --o-clustered-sequences 97_rep.qza
```
4. Generate a tree for phylogenetic diversity analyses

A rooted phylogenetic tree is needed to produce phylogenetic diversity metrics, including Faith’s Phylogenetic Diversity and weighted and unweighted UniFrac
```
# multiple alignment of the representative sequences, using mafft program

qiime alignment mafft \
  --i-sequences 97_rep.qza \
  --o-alignment aligned-rep-seqs.qza
  
# output: aligned-rep-seqs.qza
```
``` 
# remove positions that are highly variable

qiime alignment mask \
  --i-alignment aligned-rep-seqs.qza \
  --o-masked-alignment masked-aligned-rep-seqs.qza
  
# output: masked-aligned-rep-seqs.qza
```
```
# generate a phylogenetic tree from the masked alignment

qiime phylogeny fasttree \
  --i-alignment masked-aligned-rep-seqs.qza \
  --o-tree unrooted-tree.qza
  
# output: unrooted-tree.qza
```
```
# place the root of the tree at the midpoint of the longest tip-to-tip distance in the unrooted tree

qiime phylogeny midpoint-root \
  --i-tree unrooted-tree.qza \
  --o-rooted-tree rooted-tree.qza
  
# output: rooted-tree.qza
```

5. Alpha and beta diversity analysis

Computing alpha and beta diversity metrics, and generating principle coordinates analysis (PCoA) plots
```
# --p-sampling-depth 33,792, according to the lowest reads number of 84 samples

qiime diversity core-metrics-phylogenetic \
  --i-phylogeny rooted-tree.qza \
  --i-table table-cr-97.qza \
  --p-sampling-depth 33,792 \
  --m-metadata-file Metadata_16S.tsv \
  --output-dir core-metrics-results
  
# output: ./core-metrics-results/
          evenness_vector.qza
          faith_pd_vector.qza
          observed_otus_vector.qza
          rarefied_table.qza
          shannon_vector.qza
          bray_curtis_distance_matrix.qza   
          bray_curtis_pcoa_results.qza
          jaccard_distance_matrix.qza
          jaccard_pcoa_results.qza
          unweighted_unifrac_distance_matrix.qza
          unweighted_unifrac_pcoa_results.qza
          weighted_unifrac_distance_matrix.qza
          weighted_unifrac_pcoa_results.qza
          bray_curtis_emperor.qzv
          jaccard_emperor.qzv
          unweighted_unifrac_emperor.qzv
          weighted_unifrac_emperor.qzv
```
Shannon’s diversity index  (a quantitative measure of community richness)
```
# testing for associations between discrete metadata categories and alpha diversity data

qiime diversity alpha-group-significance \
--i-alpha-diversity core-metrics-results/shannon_vector.qza \
--m-metadata-file Metadata_16S.tsv  \
--o-visualization core-metrics-results/shannon-significance.qzv

# output: shannon-vector-significance.qzv
```
```
# looking at the continuous data correlation with Shannon’s index

qiime diversity alpha-correlation \
--i-alpha-diversity core-metrics-results/shannon_vector.qza \
--m-metadata-file Metadata_16S.tsv \
--o-visualization core-metrics-results/shannon-significance-association.qzv

# output: shannon-significance-association.qzv
```
Observed OTUs (a qualitative measure of community richness)
```
# testing for associations between discrete metadata categories and alpha diversity data

qiime diversity alpha-group-significance \
--i-alpha-diversity core-metrics-results/observed_otus_vector.qza \
--m-metadata-file Metadata_16S.tsv \
--o-visualization core-metrics-results/observed_otus-significance.qzv

# output: observed_otus-significance.qzv
```
```
#looking at the continuous data correlation with observed OTUs

qiime diversity alpha-correlation \
--i-alpha-diversity core-metrics-results/observed_otus_vector.qza \
--m-metadata-file Metadata_16S.tsv \
--o-visualization core-metrics-results/observed-otus-significance-association.qzv

# output: observed-otus-significance-association.qzv
```
Bray-Curtis distance (a quantitative measure of community dissimilarity)
```
# analyzing sample composition in the context of discrete metadata

qiime diversity beta-group-significance \
--i-distance-matrix core-metrics-results/bray_curtis_distance_matrix.qza \
--m-metadata-file Metadata_16S.tsv \
--m-metadata-category Group \
--o-visualization core-metrics-results/bray-curtis-group-significance.qzv \
--p-pairwise

# output: bray-curtis-group-significance.qzv
```
Unweighted UniFrac distance (a qualitative measure of community dissimilarity that incorporates phylogenetic relationships between the features)
```
qiime diversity beta-group-significance \
--i-distance-matrix core-metrics-results/unweighted_unifrac_distance_matrix.qza \
--m-metadata-file Metadata_16S.tsv \
--m-metadata-category Group \
--o-visualization core-metrics-results/unweighted-unifrac-group-site-significance.qzv \
--p-pairwise

# output: unweighted-unifrac-group-site-significance.qzv
```
Weighted UniFrac distance (a quantitative measure of community dissimilarity that incorporates phylogenetic relationships between the features)
```
qiime diversity beta-group-significance \
--i-distance-matrix core-metrics-results/weighted_unifrac_distance_matrix.qza \
--m-metadata-file Metadata_16S.tsv \
--m-metadata-category Group \
--o-visualization
core-metrics-results/weighted-unifrac-group-site-significance.qzv \
--p-pairwise

# output: weighted-unifrac-group-site-significance.qzv
```
6. Taxonomic analysis

 Greengenes 13_8 99% OTUs
```
# downlaod the Greengene database

wget -O "gg-13-8-99-515-806-nb-classifier.qza" "https://data.qiime2.org/2017.10/common/gg-13-8-99-515-806-nb-classifier.qza"
```
```
# all the OTUs can be classified as taxa

qiime feature-classifier classify-sklearn \
  --i-classifier gg-13-8-99-515-806-nb-classifier.qza \
  --i-reads 97_rep.qza \
  --o-classification taxonomy.qza
  
qiime metadata tabulate \
  --m-input-file taxonomy.qza \
  --o-visualization taxonomy.qzv
  
# output: taxonomy.qzv
```
```
# draw bar plots, level 1=k__Bacteria, level 2=k__p, level 3=k__p__c ....

qiime taxa barplot \
  --i-table table-cr-97.qza \
  --i-taxonomy taxonomy.qza \
  --m-metadata-file  Metadata_16S.tsv \
  --o-visualization taxa-bar-plots.qzv
  
# output: taxa-bar-plots.qzv
```
Exporting data
```
qiime tools extract \
  *.qza \
  --output-dir ./*
 
biom convert -i *.biom -o *.tsv --to-tsv
```

