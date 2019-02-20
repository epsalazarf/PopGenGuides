---
output:
  pdf_document: default
  html_document: default
---
# VCF to PLINK Conversion

*By Pavel Salazar-Fernandez (epsalazarf@gmail.com)*

*Human Population and Evolutionary Genomics Lab | LANGEBIO*

## About
*Documentation: [VCF [1000 Genomes]](http://www.1000genomes.org/wiki/analysis/variant%20call%20format/vcf-variant-call-format-version-41)*

This document explains how to manipulate VCF (Variant Call Format) files for their utilization in the PLINK software.

## Concatenation
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
* [FILE]**.vcf.tar.gz**

## Indexing
*Documentation: [tabix](http://www.htslib.org/doc/tabix.html)*

If a VCF file is has no accompanying TBI file or was created by merging/concatenation, an index file must be created.

Program:
* tabix

Prerequisites:
* Input data file must be position sorted and compressed.

Command:
`tabix -p vcf [VCF file]`

Output:
* [FILE].vcf.tar.gz **.tbi**

## Conversion
*Reference: [PLINK Documentation: VCF](https://www.cog-genomics.org/plink2/input#vcf)*

Program:
* plink (v1.9 or upper)

Requires:
* VCF file and its index (.tbi)

Command:

```bash
plink --vcf [VCF File] \
	--keep-allele-order \
	--vcf-idspace-to _ \
	--const-fid \
	--autosome \
	--biallelic-only strict \
	--snps-only no-DI \
	--vcf-filter \
	--vcf-half-call m \
	--vcf-require-gt \
	--make-bed \
	--out [Output file name]
```
Flags:
* `--keep-allele-order`: Maintains reference allele in the A2 position.
* `--vcf-idspace-to _`: All ID spaces will be changed to "_" (recommended).
* `--const-fid`: All samples will have the same FID (default '0').
* `--autosome`: Only autosomal sites will be included.
* `--biallelic-only strict`: Only biallelic sites present in samples will be included.
* `--snps-only no-DI`: No deletions nor insertions.
* `--vcf-filter`: Only calls that pass the quality filter will be included.
* `--vcf-half-call m`: Set half-calls to missing.
* `--vcf-require-gt`: Do not include sites with no genotype data.

Output:
* [FILE]**.bed**
* [FILE]**.bim**
* [FILE]**.fam**

## Generating PED and MAP files

Program:
* plink

Requires:
* PLINK binary file set (BED, BIM & FAM files)

Command:
`plink --bfile [BED file] --recode --tab --out [Output file name]`

Output:
* [FILE]**.ped**
* [FILE]**.map**

Note:
PED file size may be considerably much bigger than its BED equivalent.

## References
1. [Best practice for converting VCF files to plink format](http://apol1.blogspot.mx/2014/11/best-practice-for-converting-vcf-files.html)
2. [BCFtools](http://samtools.github.io/bcftools/bcftools.html)
3. [PLINK Documentation](https://www.cog-genomics.org/plink2/input#vcf)