# Pipeline of mQTL and eQTL analyses for the SalivaQTL project
In this GitHub repository, you will find the protocol elaborated by the Immunogenetics Research Lab (IRLab) from the University of the Basque Country (UPV/EHU), to perform the quality control of genotype, methylation and expression data and mQTl and eQTL analyses for the SalivaQTL project.

## Quality control of genotype data 
### 1. Variant QC
-MAF 0.01  
-HWE 1e-06  
-Call rate 0.05
```
plink
  --bfile saliva_samples \
  --maf 0.01 \
  --geno 0.05 \
  --hwe 1e-06 \
  --make-bed \
  --out saliva_samples_maf001_hwe005_hwe106
```

### 2. Sample QC
-More than 4SD from the mean of heterozigosity  
-Call rate 10%

1. Heterozigosity
1.1 Create the .imiss and .het files with plink
```
plink
  --bfile saliva_samples_maf001_hwe005_hwe106 \
  --missing \
  --het \
  --out saliva_samples_maf001_hwe005_hwe106
```
1.2 Run the [calculate_heterozigosity.R](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/calculate_heterozigosity.R) script in terminal
```
Rscript calculate_heterozigosity.R saliva_samples_maf001_hwe005_hwe106.imiss \
saliva_samples_maf001_hwe005_hwe106.het
```
1.3 Remove samples with plink
```
plink
  --bfile saliva_samples_maf001_hwe005_hwe106 \
  --remove filter-het.txt \
  --make-bed \
  --out saliva_samples_maf001_hwe005_hwe106_het
```
2. Call rate
```
plink
  --bfile saliva_samples_maf001_hwe005_hwe106_het \
  --mind 0.1 \
  --make-bed \
  --out saliva_samples_maf001_hwe005_hwe106_het_miss010
```

### 3. Prepare files for imputation
1. Create the .frq file with plink
```
plink
  --bfile saliva_samples_maf001_hwe005_hwe106_het_miss010 \
  --freq \
  --out saliva_samples_maf001_hwe005_hwe106_het_miss010
```
2. Execute [Will Rayner preimputation script](https://www.chg.ox.ac.uk/~wrayner/tools/)
```
perl HRC-1000G-check-bim.pl \
  -b saliva_samples_maf001_hwe005_hwe106_het_miss010.bim \
  -f saliva_samples_maf001_hwe005_hwe106_het_miss010.frq \
  -r 100GP_Phase3_combined.legend \
  -g \
  -p EUR \
  -o $dir_preimp \
  -v
```
```
sh Run-plink.sh
```
3. Rename and compress files 
```
for i in {1..23};
do
  vcf-sort saliva_samples_maf001_hwe005_hwe106_het_miss010-updated-chr${i}.vcf | bgzip -c > $i.vcf.gz ;
done
```
4. Impute data using 1000G as reference panel

### 4. Quality control of imputed data
-RsQ > 0.9  
-MAF > 0.01  
-HWE 0.05

1. Concat the imputed files
```
for i in {1..22};
do
echo chr${i}.dose.vcf.gz >>  tmp-concat.txt
done
```
```
echo  chrX.dose.vcf.gz >>  tmp-concat.txt
```
```
bcftools concat -f tmp-concat.txt \
--threads 16 \
-o all_chr_imputed_raw.vcf.gz \
-Oz
```
2. Filter SNPs by rsq09
```
bcftools filter -i 'R2>0.9' \
-o all_chr_imputed_rsq09.vcf \
-Ov \
all_chr_imputed_raw.vcf.gz
```
3. Filter by MAF 0.01 and HWE 0.05
```
plink
  --vcf all_chr_imputed_rsq09.vcf \
  --maf 0.01 \
  --hwe 0.05 \
  --keep-allele-order \
  --make-bed \
  --out all_chr_imputed_rsq09_maf001_hwe005
```
4. Remove sex chromosomes
```
plink \
  --bfile all_chr_imputed_rsq09_maf001_hwe005 \
  --chr 1-22 \
  --keep-allele-order \
  --make-bed \
  --out imputed_rsq09_maf001_hwe005_chr1_22
```

## Methylation data 
### Quality control  
[qc_methylation_data.R](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/qc_methylation_data.R).

#### 1. CpGs QC:
-Detection P-value < 0.01  
-Bead number 3 in at least 5% of samples per probe  
-All non-CpG probes are removed  
-SNP-related probes are removed: [Zhou et al 2016](https://academic.oup.com/nar/article-lookup/doi/10.1093/nar/gkw967) and [McCartney et al 2016](https://www.sciencedirect.com/science/article/pii/S221359601630071X?via%3Dihub)  
-Multi-hit probes are removed: [Nordlund et al 2013](https://genomebiology.biomedcentral.com/articles/10.1186/gb-2013-14-9-r105) and [McCartney et al 2016](https://www.sciencedirect.com/science/article/pii/S221359601630071X?via%3Dihub)  
-Probes located in chromosome X and Y are removed  

#### 2. Sample QC: 
-Samples with a fraction of failed probes higher tan 0.05 are removed  

#### 3. Normalization methods: 
  -Functional and Noob Normalization  
  -BMIQ Normalization  
  
#### 4. Batch correction:
  -Combat  
  
#### 5. Outliers treatment:  
  -Winsorization: pct = 0.005  

## Prepare files for mQTL analysis  

### 1. Create BED File
[create_methylation_bedfile.R](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/create_methylation_bedfile.R)

### 2. Make sure the same samples are present in the methylation and genotype files
Subset from BED and plink files  

### 3. Make sure the samples are in the same order in both files

### 4. Inverse normal transformation of methylation data  
[rnt_methylation.R](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/rnt_methylation.R)  

### 5. Covariates 
  #### 5 genetic principal components.  
  [pc_air_pc_relate.R](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/pc_air_pc_relate.R)  
  #### Age  
  #### Sex  
  #### 20 Metilation principal components 
  1. Multiple linear regression with the methylation values as the outcome, and the known confounders as predictors.  
  [pca_residuals_methylation.R](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/pca_residuals_methylation.R)  
  2. PCA with the residuals.  
  [pca_residuals_methylation.R](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/pca_residuals_methylation.R)  
  3. Remove methylation PCs associated with the genotype.  
  [gwas_mpcs.R](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/gwas_mpcs.R) 
  #### Genetic Relatedness Matrix.  
  [pc_air_pc_relate.R](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/pc_air_pc_relate.R)

### 6. Run Linear mixed model using [Apex](https://github.com/corbinq/apex?tab=readme-ov-file)  
Genotype and bed file must be indexed
```
apex cis \
--vcf imputed_samples_rsq09_maf001_hwe005_chr1_22.vcf.gz \
--bed bed_sorted.bed.gz \
--cov covariates.txt \
--kin kinship.txt \
--window 1000000 \
--threads 16 \
--long \
--prefix MQTL_LMM
```

## Quality control of expression data

### 1. Concat FASTQ files from different lanes:
[concat_lanes.sh](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/concat_lanes.sh)

### 2. Preprocessing with Fastp
[fastp.sh](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/fastp.sh)

### 3. Splice-aware mapping to genome
We used STAR (Spliced Transcripts Alignment to a Reference) software and UCSC hg19 reference genome  
[alignment.sh](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/alignment.sh)  

### 4. Transcript assembly and quantification
We used StringTie software
[stringtie.sh](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/stringtie.sh)

### 5. Sample and gene filtering and TMM transformation
-Samples with more than 5M uniquely mapped reads were selected.  
-Genes with ≥ 6 reads in ≥ 20% of samples were selected.  
-Read counts were normalized between samples using TMM
[gene_sample_qc_tmm.R](https://github.com/albahladeras/SalivaQTL_QC_genotype/blob/main/gene_sample_qc_tmm.R)

### 6. Inverse normal transformation across samples


## Prepare files for eQTL analysis  

### 1. Create BED File (columns: chr, start, end, gene_id, sample_1...sample_n)

### 2. Make sure the same samples are present in the expression and genotype files
Subset from BED and plink files  

### 3. Make sure the samples are in the same order in both files

### 4. Inverse normal transformation of expression data  

