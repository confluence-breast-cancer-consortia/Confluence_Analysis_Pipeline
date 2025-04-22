
# Regenie GWAS Pipeline for Breast Cancer Analyses in Confluence


This pipeline runs Regenie for breast cancer GWAS using imputed genotype data. For full documentation, refer to the [official Regenie guide](https://rgcgithub.github.io/regenie/).

Analyses are restricted to female participants, and only variants with minor allele count (MAC) > 30 are considered in final association steps. Initial association testing will constrain to variants with imputation quality R² > 0.2. Additional post-association QC may be applied as needed. Regenie consists of two main steps:

1. **Step 1**: Ridge regression model fitting using a subset of genotyped or imputed variants  
2. **Step 2**: Firth logistic regression-based association testing using all variants

---

## Step 1: QC for Array Data

We generate a pruned genotype dataset suitable for Step 1 of Regenie by selecting ~50,000 high-quality, LD-pruned variants.

**Filtering steps:**

- Keep variants with MAF ≥ 0.05  
- Remove variants with missingness > 5%  
- Remove samples with missing variant rate > 5%  
- LD-prune to reduce collinearity

Always filter variants first and create an updated dataset before applying sample-level missingness filters. If variant and sample filters are applied simultaneously in a single command, additional samples may be dropped due to temporary inclusion of low-quality variants.


<details>
<summary>Example Code</summary>

```bash
# Filter variants
plink2 --bfile array_data \
  --maf 0.05 --geno 0.05 \
  --make-bed --out array_data_qc_tmp \
  --threads 8 --memory 16384

# Filter samples
plink2 --bfile array_data_qc_tmp \
  --mind 0.05 \
  --write-snplist --write-samples --no-id-header \
  --make-bed --out array_data_qc \
  --threads 8 --memory 16384

# LD pruning
plink2 --bfile array_data_qc \
  --indep-pairwise 1000 100 0.9 \
  --out array_data_qc --threads 8 --memory 16384

plink2 --bfile array_data_qc \
  --extract array_data_qc.prune.in \
  --make-bed --out array_data_qc.pruned \
  --threads 8 --memory 16384
```
</details>

---

## Step 2: QC for Imputed Data

This prepares the dataset for Regenie Step 2. Commands may vary based on file format (e.g., BGEN or PLINK PGEN).

**Suggested filters:**

- Keep variants with MAC ≥ 30
- Remove variants with R² < 0.2
- Remove variants with missingness > 5%
- For homogeneous populations, exclude variants with Hardy-Weinberg Equilibrium (HWE)  P-value <1E-12
- For hetergenous populations, HWE filtering will not be applied
- Remove samples with >5% missingness  

Again, filter variants first before sample-level missingness filtering. For HWE filtering, we currently exclude variants with HWE p-values < 1E-12 in the European and East Asian ancestry groups. We define heterogeneous populations as those with recent admixture or notable population structure (e.g., African American, Latino, or some South Asian groups). Since HWE deviations in these populations may reflect structure rather than genotyping error, we skip HWE filtering for them to avoid removing valid variants.

MAC is typically defined using the combined sample of cases and controls. For ancestry-specific analyses, apply the MAC filter within the relevant stratum (e.g., MAC > 30 within each ancestry group).


---

## Step 3: Regenie Step 1: Ridge Regression

This step performs firth logistic regression testing across imputed variants. Here is the example for association testing based on CHR22. 


<details>
<summary>Example Code</summary>

```bash
regenie --step 1 \
  --bed array_data_qc.pruned \
  --extract array_data_qc.snplist \
  --keep array_data_qc.qc.id \
  --phenoFile GSA_Reformatted_Phenotypes.validated.txt \
  --phenoColList Outcome \
  --covarFile GSA_Reformatted_Covariates.validated.txt \
  --covarColList age,PC1,PC2,PC3,PC4,PC5,PC6,PC7,PC8,PC9,PC10 \
  --strict \
  --bsize 1000 --bt \
  --lowmem --lowmem-prefix tmp_rg \
  --gz --threads 8 \
  --use-relative-path \
  --out regenie_step1_out \
  --loocv
```
</details>

---

## Step 4: Regenie Step 2: Association Testing

Performs Firth logistic regression testing across imputed variants.

<details>
<summary>Example Code (CHR22 – PGEN format)</summary>

```bash
regenie --step 2 \
  --pgen chr22.pgen \
  --phenoFile GSA_Reformatted_Phenotypes.validated.txt \
  --phenoColList Outcome \
  --bsize 400 \
  --pred regenie_step1_out_pred.list \
  --threads 8 \
  --minMAC 30 --minINFO 0.2 \
  --bt --firth --approx \
  --test additive \
  --gz --no-split \
  --covarFile GSA_Reformatted_Covariates.validated.txt \
  --covarColList age,PC1,PC2,PC3,PC4,PC5,PC6,PC7,PC8,PC9,PC10 \
  --strict \
  --rare-mac 1000 \
  --out chr22_out
```
</details>

<details>
<summary>Example Code (CHR22 – BGEN format)</summary>

```bash
regenie --step 2 \
  --bgen chr22.bgen \
  --phenoFile GSA_Reformatted_Phenotypes.validated.txt \
  --phenoColList Outcome \
  --bsize 400 \
  --pred regenie_step1_out_pred.list \
  --threads 8 \
  --minMAC 30 --minINFO 0.2 \
  --bt --firth --approx \
  --test additive \
  --gz --no-split \
  --covarFile GSA_Reformatted_Covariates.validated.txt \
  --covarColList age,PC1,PC2,PC3,PC4,PC5,PC6,PC7,PC8,PC9,PC10 \
  --strict \
  --out chr22_out
```
</details>

---

## Notes
This pipeline assumes BGEN or PGEN format for imputed data. Adjust flags accordingly. QC thresholds (e.g., MAC, INFO) may be tailored based on population structure.  

