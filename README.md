# Admix_new_order_Mar_2022

Cedar is temporally out of oredr. Therefore working on graham directory,

```bash
/scratch/premacht/Admix_new_order_Mar_2022
```
I received fully filtered and finalized vcf.gz files from Ben.

```bash
all_XL_onlyXL_only_Lsubgenome.vcf.gz  
all_XL_onlyXL_only_Lsubgenome.vcf.gz.tbi
all_XL_onlyXL_only_Ssubgenome.vcf.gz  
all_XL_onlyXL_only_Ssubgenome.vcf.gz.tbi
```

I am putting them in respective folders 

```bash
mkdir l_only
mkdir s_only
mv *Lsub* l_only/
mv *Ssub* s_only/
```

then renamed vcf files inside the directories as autosomes so it is easier and we can use the same script for both l and s

in l_only,

```bash
mv all_XL_onlyXL_only_Lsubgenome.vcf.gz autosomes.vcf.gz
```
 
 same in s_only
```
mv all_XL_onlyXL_only_Ssubgenome.vcf.gz autosomes.vcf.gz
```

gunzip gz files(this in both L and S)

run this inside l_only
```
gunzip autosomes.vcf.gz
cd ../s_only
gunzip autosomes.vcf.gz
cd ../l_only
```

try filtering the data to remove positions that have >50% missing data. This might decrease the size of the data substantially. 
If the file sizes are way smaller (e.g. half as large)
then no need to thin any more

save this in l_only
```
#!/bin/sh
#SBATCH --job-name=bwa_505
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --time=24:00:00
#SBATCH --mem=32gb
#SBATCH --output=bwa505.%J.out
#SBATCH --error=bwa505.%J.err
#SBATCH --account=def-ben

module load StdEnv/2020
module load vcftools/0.1.16

vcftools --vcf autosomes.vcf --max-missing 0.5 --out ./autosomes_missing_filtered.vcf --recode

```
run in both directories

```
sbatch remove_missing_sites.sh
cp remove_missing_sites.sh ../s_only/
cd ../s_only/
sbatch remove_missing_sites.sh
cd ../l_only/
```
convert to geno format using plink , make a bed file and remove any SNP with no data for autosomes do all these for l_only and s_only seperately and we need to change the chr names in the .bim file because these cause problems for admixture: create a directory to collect outputs from next step

Run this in the directory with l_only and s_only folders

I am renaming vcf files as autosomes.vcf to make it easier for the next few steps

```bash

for i in l_only s_only; do
module load StdEnv/2020  intel/2020.1.217 bcftools/1.11
cd ${i}
mv autosomes.vcf autosomes_unfiltered.vcf
bgzip -c autosomes_missing_filtered.vcf.recode.vcf > autosomes.vcf.gz
tabix -p vcf autosomes.vcf.gz
module load nixpkgs/16.09  intel/2016.4 plink/1.9b_5.2-x86_64
plink --vcf ./autosomes.vcf.gz --make-bed --geno 0.999 --out ./autosomes --allow-extra-chr --const-fid
awk -v OFS='\t' '{$1=0;print $0}' autosomes.bim > autosomes.bim.tmp
mv autosomes.bim.tmp autosomes.bim
cd .. ; done
```







creating a directory for scripts in the same directory with l_only and s_only folders

```bash
mkdir scripts
```

now save two job arrays to cal admixture(2:6 and 7:12-run array numbers acccordingly) in scripts folder changing array numbers

as cal_admix_2to6.sh

```bash
#!/bin/sh
#SBATCH --job-name=bwa_505
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --time=48:00:00
#SBATCH --mem=16gb
#SBATCH --output=bwa505.%J.out
#SBATCH --error=bwa505.%J.err
#SBATCH --account=def-ben
#SBATCH --array=2-6

#SBATCH --mail-user=premacht@mcmaster.ca
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
#SBATCH --mail-type=REQUEUE
#SBATCH --mail-type=ALL

module load nixpkgs/16.09 admixture/1.3.0

# submitting array
 admixture --cv autosomes.bed ${SLURM_ARRAY_TASK_ID} > autosomeslog${SLURM_ARRAY_TASK_ID}.out
 ```
and cal_admix_7to13.sh

```bash
#!/bin/sh
#SBATCH --job-name=bwa_505
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --time=48:00:00
#SBATCH --mem=16gb
#SBATCH --output=bwa505.%J.out
#SBATCH --error=bwa505.%J.err
#SBATCH --account=def-ben
#SBATCH --array=7-13

#SBATCH --mail-user=premacht@mcmaster.ca
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
#SBATCH --mail-type=REQUEUE
#SBATCH --mail-type=ALL

module load nixpkgs/16.09 admixture/1.3.0

# submitting array
 admixture --cv autosomes.bed ${SLURM_ARRAY_TASK_ID} > autosomeslog${SLURM_ARRAY_TASK_ID}.out
 ```
 
 now run both of them in both subgenomes(l and s) by pasting this inside directory 'all_chrs_together_vcf'
 ```bash
 for i in l_only s_only; do  
cd ${i} 
cp ../scripts/cal_admix_2to6.sh . 
cp ../scripts/cal_admix_7to13.sh . 
sbatch cal_admix_2to6.sh 
sbatch cal_admix_7to13.sh
cd ..; done
```
ou can collect the files need to be downloaded for the next step into a new directory creating a directory called 'to_download' by running this command. Here I have excluded not needed file types to reduce the download size.

Run this in the ADMIXTURE folder

```
mkdir to_download
rsync -ap --exclude="*.vcf*" --exclude="*.sh" --exclude="bwa*" all_chrs_together_vcf/* to_download
```
 
Then download selected files

Create a sample list assigning all the samples into populations like following(here I assigned all of them to tropicalis)
you can create a tab seperated txt file with sample name in first column and species or population in second column using excel.

you can get the sample list by

```
module load StdEnv/2020  gcc/9.3.0 bcftools/1.13
bcftools query -l autosomes.vcf.gz
```
example file

```txt
2014_Inhaca_10_Inhaca_ATATGT_cuttrim_sorted.bam	Inhaca
2014_Inhaca_150_Inhaca_ATCGTA_cuttrim_sorted.bam	Inhaca
2014_Inhaca_152_Inhaca_CATCGT_cuttrim_sorted.bam	Inhaca
2014_Inhaca_24_Inhaca_CGCGGT_cuttrim_sorted.bam	Inhaca
2014_Inhaca_38_Inhaca_CTATTA_cuttrim_sorted.bam	Inhaca
2014_Inhaca_52_Inhaca_GCCAGT_cuttrim_sorted.bam	Inhaca
2014_Inhaca_65_Inhaca_GGAAGA_cuttrim_sorted.bam	Inhaca
946_Draken_TCGTT_cuttrim_sorted.bam	Draken
993_Draken_GGTTGT_cuttrim_sorted.bam	Draken
BJE3508_DeDorn_ATTGA_cuttrim_sorted.bam	DeDorn
BJE3509_DeDorn_CATCT_cuttrim_sorted.bam	DeDorn
BJE3510_DeDorn_CCTAG_cuttrim_sorted.bam	DeDorn
BJE3511_DeDorn_GAGGA_cuttrim_sorted.bam	DeDorn
BJE3512_DeDorn_GGAAG_cuttrim_sorted.bam	DeDorn
BJE3513_DeDorn_GTCAA_cuttrim_sorted.bam	DeDorn
BJE3514_DeDorn_TAATA_cuttrim_sorted.bam	DeDorn
BJE3515_DeDorn_TACAT_cuttrim_sorted.bam	DeDorn
BJE3525_Laigns_GAATTCA_cuttrim_sorted.bam	Laigns
BJE3526_Laigns_GAACTTG_cuttrim_sorted.bam	Laigns
BJE3527_Laigns_GGACCTA_cuttrim_sorted.bam	Laigns
BJE3528_Laigns_GTCGATT_cuttrim_sorted.bam	Laigns
BJE3529_Laigns_AACGCCT_cuttrim_sorted.bam	Laigns
BJE3530_Laigns_AATATGG_cuttrim_sorted.bam	Laigns
BJE3531_Laigns_ACGTGTT_cuttrim_sorted.bam	Laigns
BJE3532_Laigns_ATTAATT_cuttrim_sorted.bam	Laigns
BJE3533_Laigns_ATTGGAT_cuttrim_sorted.bam	Laigns
BJE3534_BW_CTCG_cuttrim_sorted.bam	BW
BJE3535_BW_TGCA_cuttrim_sorted.bam	BW
BJE3536_BW_ACTA_cuttrim_sorted.bam	BW
BJE3537_BW_CAGA_cuttrim_sorted.bam	BW
BJE3538_BW_AACT_cuttrim_sorted.bam	BW
BJE3539_BW_GCGT_cuttrim_sorted.bam	BW
BJE3541_BW_CGAT_cuttrim_sorted.bam	BW
BJE3542_BW_GTAA_cuttrim_sorted.bam	BW
BJE3543_BW_AGCG_cuttrim_sorted.bam	BW
BJE3544_BW_GATG_cuttrim_sorted.bam	BW
BJE3545_BW_TCAG_cuttrim_sorted.bam	BW
BJE3546_BW_TGCGA_cuttrim_sorted.bam	BW
BJE3547_GRNP_TAGGAA_cuttrim_sorted.bam	GRNP
BJE3548_GRNP_GCTCTA_cuttrim_sorted.bam	GRNP
BJE3549_GRNP_CCACAA_cuttrim_sorted.bam	GRNP
BJE3550_GRNP_CTTCCA_cuttrim_sorted.bam	GRNP
BJE3551_GRNP_GAGATA_cuttrim_sorted.bam	GRNP
BJE3552_GRNP_ATGCCT_cuttrim_sorted.bam	GRNP
BJE3553_GRNP_AGTGGA_cuttrim_sorted.bam	GRNP
BJE3554_GRNP_ACCTAA_cuttrim_sorted.bam	GRNP
BJE3573_VicW_CGCGGAGA_cuttrim_sorted.bam	VicW
BJE3574_VicW_CGTGTGGT_cuttrim_sorted.bam	VicW
BJE3575_Kimber_GTACTT_cuttrim_sorted.bam	Kimber
BJE3576_Kimber_GTTGAA_cuttrim_sorted.bam	Kimber
BJE3578_Kimber_TGGCTA_cuttrim_sorted.bam	Kimber
BJE3579_Kimber_TATTTTT_cuttrim_sorted.bam	Kimber
BJE3580_Kimber_CTTGCTT_cuttrim_sorted.bam	Kimber
BJE3581_Kimber_ATGAAAG_cuttrim_sorted.bam	Kimber
BJE3582_Kimber_AAAAGTT_cuttrim_sorted.bam	Kimber
BJE3633_Niewou_CGCTGAT_cuttrim_sorted.bam	Niewou
BJE3640_Niewou_CGGTAGA_cuttrim_sorted.bam	Niewou
BJE3641_Niewou_CTACGGA_cuttrim_sorted.bam	Niewou
BJE3642_Niewou_GCGGAAT_cuttrim_sorted.bam	Niewou
BJE3644_Niewou_TAGCGGA_cuttrim_sorted.bam	Niewou
BJE3645_Niewou_TCGAAGA_cuttrim_sorted.bam	Niewou
BJE3647_Niewou_TCTGTGA_cuttrim_sorted.bam	Niewou
BJE3654_ThreeSis_TGCTGGA_cuttrim_sorted.bam	Threesis
BJE3655_ThreeSis_ACGACTAG_cuttrim_sorted.bam	Threesis
BJE3656_ThreeSis_TAGCATGG_cuttrim_sorted.bam	Threesis
BJE3657_ThreeSis_TAGGCCAT_cuttrim_sorted.bam	Threesis
BJE3658_ThreeSis_TGCAAGGA_cuttrim_sorted.bam	Threesis
BJE3659_ThreeSis_TGGTACGT_cuttrim_sorted.bam	Threesis
BJE3660_ThreeSis_TCTCAGTG_cuttrim_sorted.bam	Threesis
BJE3661_ThreeSis_CGCGATAT_cuttrim_sorted.bam	Threesis
BJE3662_ThreeSis_CGCCTTAT_cuttrim_sorted.bam	Threesis
BJE3663_ThreeSis_AACCGAGA_cuttrim_sorted.bam	Threesis
BJE3664_ThreeSis_ACAGGGA_cuttrim_sorted.bam	Threesis
BJE3665_ThreeSis_ACGTGGTA_cuttrim_sorted.bam	Threesis
BJE3666_ThreeSis_CCATGGGT_cuttrim_sorted.bam	Threesis
BJE3667_Citrus_CGCTT_cuttrim_sorted.bam	Citrus
BJE3668_Citrus_TCACG_cuttrim_sorted.bam	Citrus
BJE3669_Citrus_CTAGG_cuttrim_sorted.bam	Citrus
BJE3670_Citrus_ACAAA_cuttrim_sorted.bam	Citrus
BJE3671_Citrus_TTCTG_cuttrim_sorted.bam	Citrus
BJE3672_Citrus_AGCCG_cuttrim_sorted.bam	Citrus
BJE3673_Citrus_GTATT_cuttrim_sorted.bam	Citrus
BJE3674_Citrus_CTGTA_cuttrim_sorted.bam	Citrus
BJE3675_Citrus_ACCGT_cuttrim_sorted.bam	Citrus
BJE3676_Citrus_GCTTA_cuttrim_sorted.bam	Citrus
BJE3677_Citrus_GGTGT_cuttrim_sorted.bam	Citrus
BJE3678_Citrus_AGGAT_cuttrim_sorted.bam	Citrus
JM_no_label1_Draken_CCACGT_cuttrim_sorted.bam	Draken
JM_no_label2_Draken_TTCAGA_cuttrim_sorted.bam	Draken
Vred_8_Vred_GCTGTGGA_cuttrim_sorted.bam	Vred
```

# Customized script

I wanted a bunch of customization in the plots given by the default R script. So I wrote the following to be used in local machine R studio.

I wrote this in a way you can just copy this script to your working folder and run. This creates a directory named plot_outs and save the resulting plots in that.

For this to work,

1)put all outputs from previous sections including all files ending with P or Q(eg; autosomes.2.p) all files created from previous scripts with different formats like .bed .bim .fam ....etc.(eg; autosomes.bed) .log and all .out files

inside each folder named- s_only and l_only(this happens automatically if you used previous scripts to select and download files)

2)sample list created with assigned populations as shown above(should be named as samples.list and the r script should be in the current working directory(not inside any of the sub genome folders)
plotADMIXTURE.r

This plots all data inside all three genomes and puts the outputs creating a folder named 'plot_outs'

