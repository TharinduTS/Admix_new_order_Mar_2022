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
convert to geno format using plink , make a bed file and remove any SNP with no data for autosomes do all these for l_only and s_only seperately and we need to change the chr names in the .bim file because these cause problems for admixture: create a directory to collect outputs from next step

Run this in the directory with l_only and s_only folders

I am renaming vcf files as autosomes.vcf to make it easier for the next few steps

```bash

for i in l_only s_only; do
module load StdEnv/2020  intel/2020.1.217 bcftools/1.11
cd ${i}
mv *subgenome.vcf.gz autosomes.vcf.gz
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
 
