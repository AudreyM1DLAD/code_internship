# code_internship
Ici, vous trouverez les codes bash effectué lors de mon stage

# bin!/bin/bash

cd Documents/Stage/
mkdir emp-paired-end-sequences

echo " "
echo Bienvenue dans l\'analyse de données PE de microbiome par Qiime 2
echo " "
echo Avant de commencer, assurez-vous que l\'environnement conda est activé
echo et que les données que vous vous apprêtez à soumettre sont au format fastq
echo " "
echo Entrez le lien de votre fichier de métadonnées
read METADATA
echo " "
echo Entrez le lien de votre fichier fastq du brin forward
read FORWARD
echo " "
echo Celui du brin reverse
read REVERSE
echo " "
echo Enfin, celui des barcodes
read BARCODES
echo Merci
echo " "

echo Téléchargement des données en cours...
wget \
  -O "sample-metadata.tsv" \
  "$METADATA"
echo ...
wget \
  -O "emp-paired-end-sequences/forward.fastq.gz" \
  "$FORWARD"
echo ...
wget \
  -O "emp-paired-end-sequences/reverse.fastq.gz" \
  "$REVERSE"
echo ...
wget \
  -O "emp-paired-end-sequences/barcodes.fastq.gz" \
  "$BARCODES"
echo ...
echo Téléchargement des données effectué
echo " "

echo 1- Import des données dans Qiime en cours...
qiime tools import \
  --type EMPPairedEndSequences \ 
  --input-path emp-paired-end-sequences \
  --output-path emp-paired-end-sequences.qza
echo Import des données dans Qiime effectué
echo " "

cd emp-paired-end-sequences/

echo Décompression des fichiers fastq en cours...
gunzip forward.fastq
gunzip reverse.fastq
gunzip barcodes.fastq
echo Décompression des fichiers fastq effectuée
echo " "

echo Génération des rapports fastqc en cours...
fastqc forward.fastq
fastqc reverse.fastq
fastqc barcodes.fastq
echo Génération des rapports fastqc effectuée
echo " "

cd ..

echo 2- Demultiplexing en cours...
qiime demux emp-paired \
  --m-barcodes-file sample-metadata.tsv \
  --m-barcodes-column barcode-sequence \
  --p-rev-comp-mapping-barcodes \
  --i-seqs emp-paired-end-sequences.qza \
  --o-per-sample-sequences 2a-demux-full.qza \
  --o-error-correction-details 2b-demux-details.qza
echo Demultiplexing effectué
echo " "

echo 3- Subsampling en cours...
qiime demux subsample-paired \
  --i-sequences 2a-demux-full.qza \
  --p-fraction 0.3 \
  --o-subsampled-sequences 3a-demux-subsample.qza
echo Subsampling effectué
qiime demux summarize \
  --i-data 3a-demux-subsample.qza \
  --o-visualization 3b-demux-subsample.qzv 
echo + Résumé visuel du subsampling
echo " "

echo 4- Trimming en cours...
qiime cutadapt trim-paired \
  --i-demultiplexed-sequences 3a-demux-subsample.qza \
  --p-cores 3 \
  --o-trimmed-sequences 4-demux-trim-subsample.qza
echo Trimming effectué
echo " "

echo Préparation pour le filtering...
qiime tools export \
  --input-path 3b-demux-subsample.qzv \
  --output-path ./demux-subsample/

echo 5- Filtering en cours...
qiime demux filter-samples \
  --i-demux 4-demux-trim-subsample.qza \
  --m-metadata-file ./demux-subsample/per-sample-fastq-counts.tsv \
  --p-where 'CAST([forward sequence count] AS INT) > 100' \
  --o-filtered-demux 5-demux-filter-trim-subsample.qza
echo Filtering effectué 
echo " "

echo 6- Merging en cours...
qiime vsearch merge-pairs \
  --i-demultiplexed-seqs 5-demux-filter-trim-subsample.qza \
  --p-threads 2 \
  --o-merged-sequences 6-demux-merge-filter-trim-subsample.qza
echo Merging effectué 
echo " "

echo 7- Contrôle Qualité en cours...
qiime quality-filter q-score \
  --i-demux 6-demux-merge-filter-trim-subsample.qza \
  --o-filtered-sequences 7a-postCQ-sequences-demux-merge-filter-trim-subsample.qza \
  --o-filter-stats 7b-CQ-stats.qza
echo Contrôle Qualité effectué 
echo " "

echo 8- Déreplication des séquences en cours...
qiime vsearch dereplicate-sequences \
  --i-sequences 7a-postCQ-sequences-demux-merge-filter-trim-subsample.qza \
  --o-dereplicated-table 8a-table-dereplicated-seq-postCQ.qza \
  --o-dereplicated-sequences 8b-dereplicated-seq-postCQ.qza
echo Déreplication des séquences effectuée
echo " "

echo 9- Clustering en OTUs en cours...
qiime vsearch cluster-features-de-novo \
  --i-sequences 8b-dereplicated-seq-postCQ.qza \
  --i-table 8a-table-dereplicated-seq-postCQ.qza \
  --p-perc-identity 0.97 \
  --p-threads 3 \
  --o-clustered-table 9a-clustered-table.qza \
  --o-clustered-sequences 9b-clustered-seq.qza
echo Clustering en OTUs effectué
echo " "


qiime feature-table summarize \
  --i-table 9a-clustered-table.qza \
  --o-visualization 9c-OTUs-table.qzv 
echo + Résumé visuel du clustering
echo " "

qiime tools export \
  --input-path 9a-clustered-table.qza \
  --output-path 9d-table.biom
echo Export de la feature-table 

cd 9d-table.biom/ 
biom convert \
  -i feature-table.biom \
  -o 9e-table_OTUs.tsv \
  --to-tsv  
echo " "
echo Le Clustering est terminé, les résultats sont accessibles dans une feature-table au format.tsv 
echo " "

cd ..
echo 10- Denoising ASVs en cours...
qiime dada2 denoise-paired \
  --i-demultiplexed-seqs 2a-demux-full.qza \
  --p-trim-left-f 13 \
  --p-trim-left-r 13 \
  --p-trunc-len-f 150 \
  --p-trunc-len-r 150 \
  --o-table 10a-table.qza \
  --o-representative-sequences 10b-rep-seqs-denoising-ASVs.qza \
  --o-denoising-stats 10c-denoising-stats.qza
echo Denoising ASVs effectué 
echo " "


qiime tools export \
  --input-path 10a-table.qza \
  --output-path 10d-table.biom
echo Export de la feature-table 

cd 10d-table.biom/ 
biom convert \
  -i feature-table.biom \
  -o 10e-table_OTUs.tsv \
  --to-tsv  
echo " "
echo Le Denoising est terminé, les résultats sont accessibles dans une feature-table au format.tsv 
echo " "

cd ..
# Taxonomie non fonctionnelle
echo 11- Classification taxonomique en cours...
qiime feature-classifier classify-sklearn \
  --i-classifier gg_2022_10_backbone_full_length.nb.qza \
  --i-reads 9b-clustered-seq.qza \
  --p-reads-per-batch 100 \
  --p-n-jobs 4 \
  --o-classification 11a-taxonomy.qza
echo " " 
echo Classification taxonomique effectuée

    
qiime tools export \
  --input-path 12-filtered-table.qza \
  --output-path 12-table.biom
echo Export de la feature-table 


cd 12-table.biom/ 
biom convert \
  -i feature-table.biom \
  -o 12-filtered-table_OTUs.tsv \
  --to-tsv  
 
# Adaptation de la commande de taxonomie non fonctionnelle
qiime feature-classifier classify-sklearn \
  --i-classifier gg_2022_10_backbone_full_length.nb.qza \
  --i-reads 12-filtered-table.qza \
  --p-reads-per-batch 100 \
  --p-n-jobs 4 \
  --o-classification 11a-taxonomy.qza

# Réduction des données pour espérer faire fonctionner la taxonomie.

qiime feature-table filter-features-conditionally \
    --i-table 9a-clustered-table.qza \
    --p-abundance 0.01 \
    --p-prevalence 0.10 \
    --o-filtered-table 12-filtered-table.qza

qiime feature-table filter-seqs \
  --i-data 9b-clustered-seq.qza \
  --i-table 12-filtered-table.qza \
  --o-filtered-data 13-filtered-seqs.qza
  
  



