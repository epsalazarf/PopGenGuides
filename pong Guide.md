# pong Tutorial

*By Pavel Salazar-Fernandez (epsalazarf@gmail.com)*

*Human Population and Evolutionary Genomics Lab | LANGEBIO*

## About
*Documentation: [pong (GitHub)](https://github.com/abehr/pong)*

This document explains how to generate multiple ADMIXTURE data for integration in **pong**.

## pong
>pong is a package consisting of a new algorithm for fast post-hoc analysis of clustering inference output from population genetic data and a custom, interactive D3.jsbased data visualization. Our package solves the challenges of accounting for label switching, characterizing multiple modes, and aligning Q matrices across values of K by constructing weighted bipartite graphs for each pair of Q matrices depicting similarity in membership coefficients between clusters.

For installation instructions, an introductory tutorial and the detailed manual, check the software web page:

<https://github.com/abehr/pong>

## Procedure

**Requires:**
- [PLINK](http://pngu.mgh.harvard.edu/~purcell/plink/) (version 1.07+)
- [ADMIXTURE](https://www.genetics.ucla.edu/software/admixture/)
- [pong](https://github.com/abehr/pong)

1. Get the PLINK binary files (.bed, .bim, .fam) to be analyzed. The dataset must contain only the samples of interest. Preferably, replace the samples' `FAMID`s with their population tags in the `.fam` file for easier grouping and sorting.

2. Run the multiple ADMIXTURE analysis with the following command (script is annexed at the end of this document):
`multiAdmix.sh [DATASET] [minimum K] [maximum K] [iterations]`
Be warned that depending on the number of samples, K's and iterations, this analysis can become very long.

3. When all the analyses finishes, the results will be saved at a directory called `[DATASET]-multiAdmix/`, containing multiple `run[##]/` directories filled with  Q files at each specified K.

4. Generate the following files required by **pong** at the `[DATASET]-multiAdmix/` directory:
   - `pong_filemap.txt`: a list containing the location for all the Q files to be integrated by pong. The `pongmap.sh` can generate it automatically by running it in the "[DATASET]-multiAdmix/" directory (script included in the annex).
   - `ind2pop.txt`: a list with repeated populations tags used to replace the individual tags; must contain the same number of lines as samples. If you replaced the FAMID with populations tags, just run `cut -f 1 [DATASET].fam > ind2pop.txt`.
   - `pop_order.txt`: used to sort the populations in the display.
   - `pop_order_expanded.txt`: same as `pop_order.txt`, with a second column for extended names to replace the population tags in the plots.

5. Run **pong** in the `[DATASET]-multiAdmix/` directory with the following command:
`pong -m pong_filemap -n pop_order_expandednames.txt -i ind2pop.txt`

6. Once finished, a port for visualizing the results will start. Open a web browser (not IE) and go to the address <http://localhost:4000>.

### Other functions

TBD

## Scripts
### multiAdmix.sh
```
#!/bin/bash

#MULTIADMIX.sh
# $1: Dataset name
# $2: Min K
# $3: Max K
# $4: Iterations

echo "Starting MultiAdmix..."

if [[ $2 = [0-9]* && $3 = [0-9]* && $4 = [0-9]* ]] && (($2 <= $3))

then
  echo "Dataset: " $1
  echo "Min K: " $2
  echo "Max K: " $3
  echo "Iterations: " $4

  mkdir $1-multiAdmix
  cd $1-multiAdmix
  for ((i=1; i<=$4; i++))
  do
    echo "Run " $i
    mkdir run$i
    x=$RANDOM
    cd run$i
    for ((k=$2; k<=$3; k++))
    do
      admixture --cv -s $x ../../${1}.bed $k > ${1}.r${i}k${k}.log
    done
    cd ..
  done

  echo "All runs finished!"
else
  echo "ERROR: non-integer in input parameters"
  echo "Usage: multiAdmix.sh {dataset} {minK} {maxK} {iterations}"
fi

#END

```

### pongmap.sh
```
#!/bin/bash

runs=$(find -name "run*" | sed 's/\.\/run\([0-9][0-9]*\)/\1/' | sort -n | tail -n 1)
mink=$(find -name "*.Q" | rev | cut -f 2 -d "." | sort  -n | uniq | head -n 1)
maxk=$(find -name "*.Q" | rev | cut -f 2 -d "." | sort  -n | uniq | tail -n 1)
root=$(find -name "*.Q" | rev | cut -f 1  -d "/" | cut -f 3- -d "." | rev | head -n 1)

echo "Files name: " $root
echo "Runs found: " $runs
echo "MinK found: " $mink
echo "MaxK found: " $maxk

echo "Starting listing..."

touch pong_filemap.txt

for ((k=$mink; k<=$maxk; k++))
do
  echo "K:" $k
  for ((r=1; r<=$runs; r++))
  do
    echo -e "k${k}r${r}\t$k\trun${r}/${root}.${k}.Q" >> pong_filemap.txt
  done
done

echo "List finished! > pong_filemap.txt"

#END

```
___
