# Description of Edits

I very lightly edited the original [LDpred](https://github.com/bvilhjal/ldpred) scripts for my own use, mainly to fix a few bugs (potentially will make future pull requests), force the BASIC input format to accept effect allele weights instead of OR's, save out a list of overlapping SNPs, auto-exclude the X chromosome from analyses, and output the final betas both in terms of A1. Line-by-line descriptions of changes can be found in DIFF_coord_genotypes.log and [DIFF_LDpred_inf.log](https://github.com/ce-carey/ldpred/blob/master/ldpred/DIFF_LDpred_inf.log). I have not edited the main `LDpred.py` script--just the infinitesimal model. For information on how to use these scripts, please see the original [README](https://github.com/bvilhjal/ldpred/blob/master/README.md), reprinted below. 

NOTE: This is under active development and generally intended for my own use and the use of my collaborators. USE AT YOUR OWN PERIL!!

## Input

The style of the edited BASIC input format to `coord_genotypes.py` is as follows:

CHR | SNP | A1 | A2 | BP | WEIGHT | P       
--- | --- | --- | --- | --- | --- | ---
chr1 | rs4951859 | C | G | 729679 | -0.02170 | 0.2083  
chr1 | rs142557973 | T | C | 731718 | 0.01930 | 0.3298  

The main difference between the above-described format and the original LDpred BASIC format is that I've included an effect allele weight rather than an OR. As such, all OR's in the original sumstats file must be converted to log(OR) prior to input. Please use the code in my [process-sumstats repo](https://github.com/ce-carey/process-sumstats), forked from [LDSC](https://github.com/bulik/ldsc) to convert a variety of sumstats files into this input format. I have altered this input, and correspondingly my code, to use a weight rather than an OR in order to allow for a greater variety of sumstat inputs (e.g., both OR's from case/control studies and betas from studies of continuous traits). 

The output of `coord_genotypes.py` forms the input to `LDpred_inf.py`. 

## Output

The main output of `LDpred_inf.py` is a file of variants and their LDpred-adjusted weights/betas:

chrom | pos | sid | nt1 | nt2 | raw_beta | ldpred_inf_beta
--- | --- | --- | --- | --- | --- | ---
chrom_1 | 754182 | rs3131969 | A | G | 3.2303e-02 | 1.9363e-04
chrom_1 | 779322 | rs4040617 | G | A | 2.8595e-02 | 1.5704e-04

In contrast to the [orignal code](https://github.com/bvilhjal/ldpred), I have flipped the sign of `ldpred_inf_beta` so that it's reported in terms of nt1 (A1 in the input). In the original code, `raw_beta` is reported in terms of nt1/A1, but `ldpred_inf_beta` is reported in terms of nt2/A2. I changed the latter to be consistent both with the `raw_beta` and the convention of various polygenic scoring software, which tend to assign weights to A1. 

## Sample Workflow

1. Convert summary stats into the adapted LDpred BASIC format (see above) using [process-sumstats](https://github.com/ce-carey/process-sumstats).

2. Run `coord_genotypes.py` and `LDpred_inf.py` together using the following script. This script runs `coord_genotypes.py`, automatically calculates the radius parameter from the total number of SNPs in common between the reference file, sumstats file, and target file, and then runs `LDpred_inf.py`:
```
reffile='/path/to/reference/genome'
sumstats='/path/to/preformatted/sumstats'
sampsize=SAMPSIZE
outname='OUTNAME'
target='/path/to/bim/file/of/target/sample/snps'

./coord_genotypes.py --gf=$reffile --ssf=$sumstats --N=$sampsize --out=$outname --vbim=$target --ssf_format="BASIC" --maf=0.00

nsnps=$(wc -l $outname.snplist | awk '{print $1}')
radius=$((nsnps/3000))

./LDpred_inf.py --coord=$outname.coord --ld_radius=$radius --ld_prefix=$outname"_LD" --N=$sampsize --out=$outname"_LDPred"
```
   
3. Score the target sample in a program such as [PLINK2](https://www.cog-genomics.org/plink2/) or [Hail](https://hail.is/) using the output of `LDpred_inf.py`.

## Tips, Tricks, and Gotcha's

Running LDpred can be very computationally resource-intensive. Below are some tips for getting it running smoothly, with some specificity to the Broad cluster and UGER.

* Typically I've used the [1000 Genomes](http://www.internationalgenome.org/) EUR subset as a reference genome. I've found that most GWAS summary statistics I use come from data imputed using 1KG. On the Broad cluster, provided you're a member of the ATGU group, you can access the latest version of the data (1KG Phase 3 version 5a), here: `/humgen/atgu1/fs03/shared_resources/1kG/integrated/20130502/`.

* Filter your reference panel and target SNP list to only the [HM3](https://www.sanger.ac.uk/resources/downloads/human/hapmap3.html) SNPs. These are SNPs that are generally well imputed across arrays and representative across the genome. A list of HM3 SNPs can be found here on the Broad cluster, provided you're a member of the ATGU group: `/humgen/atgu1/fs03/shared_resources/ldsc_reference/w_hm3.snplist`. Otherwise you can generate your own list by downloading the above-linked files.

* Using the HM3-filtered 1KG EUR reference panel and ~1 million SNPs in the sumstats and target files, I've been able to run both `coord_genotypes.py` and `LDpred_inf.py` using "only" 64GB RAM in under 2 hours. 
