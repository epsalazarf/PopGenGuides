# TreeMix Tutorial

*By Erika Landa & Pavel Salazar-Fernandez (epsalazarf@gmail.com)*

*Human Population and Evolutionary Genomics Lab | LANGEBIO*

## About
*Documentation: [TreeMix](http://bitbucket.org/nygcresearch/treemix/wiki/Home)*

This document explains how to manipulate PLINK data for a TreeMix analysis and plotting.

## Preparing the Data

**Requires:**
- PLINK (version 1.07+)
- Python 2.7+

1. Get the PLINK binary files (.bed, .bim, .fam) to be analyzed. The dataset must contain only samples of interest.
2. Generate a population tags file containing three columns: FAMID and ID (the same as the samples from the plink files), and an added column **POP** that tags each sample to a population. If the FAMID is the POP tag, you can generate thepopulation tags file like this:
`awk '{print $1,$2,$1}' DATA.bim > poptags.txt`
Example (without header):
*FAMID* *ID* *POP*
fam1  ind1  popA
fam2  ind2  popA
fam3  ind3  popA
fam4  ind4  popB
fam5  ind5  popB
... ... ...

3. Run the following command to generate a cluster file (.clst) with PLINK:
`plink --bfile [DATA] --within poptags.txt --write-cluster --out clusterstep`

4. To generate an stratified frequency file, run:
`plink --bfile [DATA] --freq --missing --within clusterstep.clst --out [DATA]`

5. Compress the resulting file into a gzip format:
`gzip [DATA].frq.strat`

6. Run the given script `plink2treemix.py` to convert the dataset to a TreeMix input file:
`python plink2treemix.py [DATA].frq.strat.gz [DATA].tm.in.gz`

## Performing the Analysis
1. Having the input file ready, a basic maximum-likelihood tree can be generated:
`treemix -i [DATA].tm.gz -o [DATA].tm.m0`
2. To account for migration events, a `-m` parameter followed by the number of migrations allowed can be added to the basic command.
`treemix -i [DATA].tm.gz -m 1 -o [DATA].tm.m1`
`treemix -i [DATA].tm.gz -m 2 -o [DATA].tm.m2`
`treemix -i [DATA].tm.gz -m 3 -o [DATA].tm.m3`

## Plotting the Results
1. Load the given `plotting_funcs2.R` script inside an R environment.
`source("plotting_funcs2.R")`
2. Use the `plot_tree` function to generate the graphics:
`plot_tree("[DATA].tm.m0")`

## Auto-Mode Script
The following script performs all previous steps automatically. It requires that the FAMIDs are already replaced by POP tags. The max number of migrations can also be set (default: 3).
```bash
[Work in progress]
```
---
