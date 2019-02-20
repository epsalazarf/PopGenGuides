# Beagle3 to VCF Conversion

*By Pavel Salazar-Fernandez (epsalazarf@gmail.com)*

*Human Population and Evolutionary Genomics Lab | LANGEBIO*

## About
*Documentation: [BEAGLE 3](http://faculty.washington.edu/browning/beagle/b3.html), [VCF [1000 Genomes]](http://www.1000genomes.org/wiki/analysis/variant%20call%20format/vcf-variant-call-format-version-41)*

This document explains how to convert a PHASED file (Beagle3) to VCF (Variant Call Format).

*WARNING*: The conversion will **de-phase** the input file.

## Conversion
### PHASED to VCF

Program:
* [fcGENE](http://sourceforge.net/projects/fcgene/files/) (v1.0.7 or upper)

Requires:
* PHASED file for each chromosome

Command:
`fcgene --bgl [FILE].phased --oformat vcf --out [OUTPUT]`

Output:
* [FILE]**.vcf**

*WARNING*: This conversion has some bugs, **the resulting VCF file must be fixed before using it**.

### Fixing the header
The output VCF may have some errors in its header that will not be allowed for some software. Use the following commands to correct them.

* Missing: contig line
`awk '/##source=fcGENE/ { print; print "##contig=<ID=[chr]",assembly=b37>"; next }1' [FILE].vcf > [FILE].fix.vcf`

* Switched FORMAT and FILTER tags:
`sed -i 's/##FILTER=<ID/##FORMAT=<ID/1' [FILE].fix.vcf`
`sed -i 's/##FORMAT=<ID=PASS/##FILTER=<ID=PASS/1' [FILE].fix.vcf`

Output:
* [FILE].**fix**.vcf

### Adding position info
Since PHASED does not save any position info, the chromosome and position must be re-supplied to the generated VCF file. This requires a SNP list.

#### SNP List
*Source: [dbSNP FTP readme](ftp://ftp.ncbi.nlm.nih.gov/snp/specs/00readme.txt)*

A table containing a list of SNPs for each chromosome with their position. This information can be downloaded from dbSNP:

[dbSNP: human, build 147, GRCh37](ftp://ftp.ncbi.nih.gov/snp/organisms/human_9606_b147_GRCh37p13/chr_rpts/)

The downloaded files can be reduced to simpler table like this one:

```
1   1000  rs123
1   2000  rs456
1   3000  rs789
```

Command:
`awk '{if($8 ~ /^NT_/ && $12 > 0)print$7,$12,"rs"$1 | "sort -n -k2"}' OFS="\t" [chr].txt > [chr].cprs.txt`

#### Procedure

Program:
* awk

Requires:
* Converted VCF file
* SNP list per chromosome (see above)

Command:
`awk 'NR==FNR{a[$3]=$2;next}($1 ~ /^#/){print;next}($3 in a){$1=[chr];$2=a[$3];print | "sort -n -k2"}' FS="\t" OFS="\t" [chr].cprs.txt [FILE].fix.vcf > [FILE].cprs.vcf`

* `NR==FNR{a[$3]=$2;next}`: Generates index from first file.
* `($1 ~ /^#/){print;next}`: Prints headers without any modification.
* `($3 in a){$1=[chr];$2=a[$3];print | "sort -n -k2"}`: If the rsID was found in the index, modifies the CHROM and POS columns, and then sorts SNPs by position in increasing order.

*Warning*: this command will erase any SNP left with no position info.

Output:
* [FILE].**cprs**.vcf


#### Auto-conversion
This script loops through all the autosomic chromosome files and fixes the beagle outputs. Customize it for your needs:

```bash
for i in {1..22}; do
  printf "\nConverting chr$i ...\n"
  ./fcgene --bgl [file].chr$i.phased --oformat vcf --out [file].chr$i.raw
  gunzip [file].chr$i.raw_vcf.gz
  mv [file].chr$i.raw_vcf [file].chr$i.raw.vcf
  rm [file].chr$i.raw_*
  
  printf "\nFixing chr$i ...\n"
  awk -v chr=$i '/##source=fcGENE/ { print; print "##contig=<ID="chr",assembly=b37>"; next }1' "[file].chr$i.raw.vcf" > temp.vcf
  sed -i 's/##FILTER=<ID/##FORMAT=<ID/1' temp.vcf
  sed -i 's/##FORMAT=<ID=PASS/##FILTER=<ID=PASS/1' temp.vcf
  awk -v chr=$i 'NR==FNR{a[$3]=$2;next}($1 ~ /^#/){print;next}($3 in a){$1=chr;$2=a[$3];print | "sort -n -k2"}' FS="\t" OFS="\t" "chr_$i.cprs.txt" temp.vcf > "[file].chr$i.fix.vcf"
  rm temp.vcf
  
  printf "\nFinished chr$i ...\n"

done
```
## Subset VCF
After converting from Beagle3 to VCF and , software designed for

Program:
* [bcftools](http://samtools.github.io/bcftools/bcftools.html#view)

Requirements:
* VCF file
* List of sample IDs

Command:
`bcftools view [FILE].vcf -S samples.ids -o [FILE].sub.vcf -M2 -c 2:minor -O v` 

* `view`: bcftools mode used for extracting data.
* `-S [file]`: reads a list of IDs (one ID per line) and keepsonly those individuals.
* `-M2`: Only outputs biallelic SNPs.
* `-c 2:minor` Discards monomorphic SNPs (minor allele count = 0) and singletons (minor allele count = 1)
* `-O`: Outputs to VCF

Output:
[FILE].**sub**.vcf

## Re-phasing

### Basic Phasing 
The basic mode of `SHAPEIT`, fast and requires little input, but more prone to errors.

Requirements:

* Converted VCF file (fixed and with SNP positions)
* [Genetic map](http://mathgen.stats.ox.ac.uk/genetics_software/shapeit/shapeit.html#gmap)

Command: 
`shapeit -V [FILE].cprs.vcf -M genetic_map.[chr].txt -O [FILE].ph.vcf`

* `-V`: VCF file to be phased
* `-M`: Genetic map file for that chromosome
* `-O`: Output name

Output:
* [FILE].**phs**.haps
* [FILE].**phs**.sample

#### Convert to VCF
To output `SHAPEIT` results to a VCF (again), use: 

`shapeit -convert --input-haps [FILE] --output-vcf [FILE].vcf`

### Phasing with Reference
*Source: [SHAPEIT: Reference](http://mathgen.stats.ox.ac.uk/genetics_software/shapeit/shapeit.html#reference)*

Since the output VCF lost all phasing information during the conversion, it is necessary to re-phase the data using a reference panel.

Requirements:
* [Genetic map](http://mathgen.stats.ox.ac.uk/genetics_software/shapeit/shapeit.html#gmap)
* [Reference genotypes (VCF)](http://mathgen.stats.ox.ac.uk/impute/1000GP_Phase3.html)
* [Reference legend & sample](http://mathgen.stats.ox.ac.uk/genetics_software/shapeit/shapeit.html#haplegsample)
* Converted VCF file with positions

*NOTE:* check all the data is in the same reference genome build (NCBI b37)
 
It is recommended to test the files before attempting the phasing.

Command:
`shapeit -check --input-vcf [FILE].cprs.vcf -M genetic_map.txt --input-ref reference.haplotypes.gz reference.legend.gz reference.sample --output-log [FILE].aligments`

#### Alignment Errors
`SHAPEIT` will produce an error warning when:
* The input file and the reference have incompatible alleles (strand issues).
* The input file has SNPs not present in the reference.

Two log files will be created:
* `myLogFile.snp.strand`: describes in detail all the problems found.
* `myLogFile.snp.strand.exclude`: a list of the physical positions of all problems 

If this problem occurs, add to the command above the following flag:
`--exclude-snp myLogFile.snp.strand.exclude`

#### Using specific groups

`echo 'XYZ' > grp.list`

#### Start Reference Phasing
If the files pass the previous tests, start the phasing process.

Command:
`shapeit --input-vcf [FILE].cprs.vcf -M genetic_map.txt --input-ref reference.haplotypes.gz reference.legend.gz reference.sample -O [FILE].refphs`

Optional flags:
* `--exclude-snp myLogFile.snp.strand.exclude`
* `--include-grp grp.list` or `--include-grp grp.list`


Output:
* [FILE].**refphs**.haps
* [FILE].**refphs**.sample

#### Convert to VCF
To output `SHAPEIT` results to a VCF (again), use: 

`shapeit -convert --input-haps [FILE] --output-vcf [FILE].vcf`

### Concatenation
*Documentation: [bcftools concat](https://samtools.github.io/bcftools/bcftools.html#concat)*

If the dataset is split in two or more VCF files (e.g. in chromosomes), it is necessary to concatenate them to create a single fused file. This procedure will create a compressed VCF file (.vcf.gz).

Program:
* bcftools

Prerequisites:
* All source files must have the same sample columns appearing in the same order.
* Input files must be sorted by chromosome and positions.
* Files must be given in the correct order.
* (Optional) A file containing a ordered list of the input files

Command:
`bcftools concat -f [VCF files list] -o [Output file name] -O z`

Output:
* [FILE]**.vcf.gz**

---
