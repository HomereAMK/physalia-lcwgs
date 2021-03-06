
Here you will learn how to perform a scan for positive selection by calculating several summary statistics in windows from low-depth NGS data.

Specifically, you will learn how to estimate:

1. site frequency spectrum
2. population genetic differentiation
3. nucleotide diversity

Please make sure to follow the preparatory instructions on the main page before running these examples.
```
NGS=/programs/angsd0.930/
DIR=/workdir/arne/physalia_lcwgs_data/data_practicals/
DATA=$DIR/BAMS/
REF=$DIR/Ref.fa
ANC=$DIR/outgrp_ref.fa

mkdir Results
mkdir Data
```

As reference, these are the labelling for each population:
- JIGA: Jekyll Island, Georgia
- PANY: Patchogue, New York
- MBNS: Minas Basin, Nova Scotia
- MAQU: Magdalen Island, Quebec

Here we see how to compute the 2D-SFS, FST, PBS, and other summary statistics from low-depth data using ANGSD.
Our final goal is to detect signatures of selection in our data.

-----------------------------

#### 1. Site frequency spectrum

One of the most important aspect of data analysis for population genetics is the estimate of the Site Frequency Spectrum (SFS).
SFS records the proportions of sites at different allele frequencies. It can be folded or unfolded, and the latter case implies the use of an outgroup species to define the ancestral state.
SFS is informative on the demography of the population or on selective events (when estimated at a local scale).

We use ANGSD to estimate SFS using on example dataset, using the methods described [here](http://www.ncbi.nlm.nih.gov/pubmed/22911679).
Details on the implementation can be found [here](http://popgen.dk/angsd/index.php/SFS_Estimation).
Briefly, from sequencing data one computes genotype likelihoods (as previously described).
From these quantities ANGSD computes posterior probabilities of Sample Allele Frequency (SAF), for each site.
Finally, an estimate of the SFS is computed.

These steps can be accomplished in ANGSD using `-doSaf 1/2` options and the program `realSFS`.

```
$NGS/angsd/angsd -doSaf
...
-doSaf		0
	1: perform multisample GL estimation
	2: use an inbreeding version
	3: calculate genotype probabilities (use -doPost 3 instead)
	4: Assume genotype posteriors as input (still beta) 
	-doThetas		0 (calculate thetas)
	-underFlowProtect	0
	-fold			0 (deprecated)
	-anc			(null) (ancestral fasta)
	-noTrans		0 (remove transitions)
	-pest			(null) (prior SFS)
	-isHap			0 (is haploid beta!)
	-doPost			0 (doPost 3,used for accesing saf based variables)
NB:
	  If -pest is supplied in addition to -doSaf then the output will then be posterior probability of the sample allelefrequency for each site
```

The SFS is typically computed for each population separately. 

We cycle across all populations and compute SAF files. By specifying an outgroup genome using the `-anc $ANC` option, we can polarize the sample allele frequencies and estimate the derived allele frequency. This will allow us to later compute the unfolded site frequency spectrum.
```
for POP in JIGA PANY MBNS MAQU
do
        echo $POP
        $NGS/angsd/angsd -b $DIR/$POP'_bams.txt' -ref $REF -anc $ANC -out Results/$POP \
                -uniqueOnly 1 -remove_bads 1 -only_proper_pairs 1 -trim 0 -C 50 \
                -minMapQ 20 -minQ 20 -minInd 5 -setMinDepth 7 -setMaxDepth 30 -doCounts 1 \
                -GL 1 -doSaf 1
done
```
Please ignore various warning messages.

```
$NGS/angsd/misc/realSFS print Results/PANY.saf.idx | less -S  
```
These values represent the sample allele frequency likelihoods at each site, as seen during the lecture.
So the first value (after the chromosome and position columns) is the likelihood of having 0 copies of the derived allele, the second indicates the probability of having 1 copy and so on.
Note that these values are in log format and scaled so that the maximum is 0.

**QUESTION**
Can you spot any site which is likely to be variable (i.e. polymorphic)?

In fact, SNPs are represented by sites where the highest likelihood does not correspond to allele frequencies of 0 or 100%.

The next step would be to use these likelihoods and estimate the overall SFS.
This is achieved by the program `realSFS`.
```
$NGS/angsd/misc/realSFS
-> ---./realSFS------
	-> EXAMPLES FOR ESTIMATING THE (MULTI) SFS:

	-> Estimate the SFS for entire genome??
	-> ./realSFS afile.saf.idx 

	-> 1) Estimate the SFS for entire chromosome 22 ??
	-> ./realSFS afile.saf.idx -r chr22 

	-> 2) Estimate the 2d-SFS for entire chromosome 22 ??
	-> ./realSFS afile1.saf.idx  afile2.saf.idx -r chr22 

	-> 3) Estimate the SFS for the first 500megabases (this will span multiple chromosomes) ??
	-> ./realSFS afile.saf.idx -nSites 500000000 

	-> 4) Estimate the SFS around a gene ??
	-> ./realSFS afile.saf.idx -r chr2:135000000-140000000 

	-> Other options [-P nthreads -tole tolerence_for_breaking_EM -maxIter max_nr_iterations -bootstrap number_of_replications]

	-> See realSFS print for possible print options
	-> Use realSFS print_header for printing the header
	-> Use realSFS cat for concatenating saf files

	->------------------
	-> NB: Output is now counts of sites instead of log probs!!
	-> NB: You can print data with ./realSFS print afile.saf.idx !!
	-> NB: Higher order SFSs can be estimated by simply supplying multiple .saf.idx files!!
	-> NB: Program uses accelerated EM, to use standard EM supply -m 0 
	-> Other subfunctions saf2theta, cat, check, dadi
```

Therefore, this command will estimate the SFS for each population separately:
```
for POP in JIGA PANY MBNS MAQU
do
        echo $POP
        $NGS/angsd/misc/realSFS Results/$POP.saf.idx > Results/$POP.sfs
done
```
The output will be saved in `Results/POP.sfs` files.

You can now have a look at the output file, for instance for the JIGA samples:
```
cat Results/JIGA.sfs
```
The first value represents the expected number of sites with derived allele frequency equal to 0, the second column the expected number of sites with frequency equal to 1 and so on.


**QUESTION**

How many values do you expect?

```
awk -F' ' '{print NF; exit}' Results/JIGA.sfs
```
Indeed this represents the unfolded spectrum, so it has `2N+1` values with N diploid individuals.

Why is it so bumpy?

The maximum likelihood estimation of the SFS should be performed at the whole-genome level to have enough information.
However, for practical reasons, here we could not use large genomic regions.
Also, as we will see later, this region is not really a proxy for neutral evolution so the SFS is not expected to behave neutrally for some populations.
Nevertheless, these SFS should be a reasonable prior to be used for estimation of summary statistics.

Optionally, one can even plot the SFS for each pop using this simple R script.
```
Rscript $NGS/Scripts/plotSFS.R Results/JIGA.sfs-Results/PANY.sfs-Results/MBNS.sfs-Results/MAQU.sfs JIGA-PANY-MBNS-MAQU 0 Results/ALL.sfs.pdf
evince Results/ALL.sfs.pdf
```

Do they behave like expected?

Which population has more SNPs?

Which population has a higher proportion of common (not rare) variants?


---------------------------------------


**Optional: The folded minor allele frequency spectrum**
If you do not have an outgroup genome, you can compute the folded site frequency spectrum by estimating sample allele frequencies by specifying the reference genome as the ancestral genome: 
```
for POP in JIGA PANY MBNS MAQU
do
        echo $POP
        $NGS/angsd/angsd -b $DIR/$POP'_bams.txt' -ref $REF -anc $REF -out Results/$POP.folded \
                -uniqueOnly 1 -remove_bads 1 -only_proper_pairs 1 -trim 0 -C 50 \
                -minMapQ 20 -minQ 20 -minInd 5 -setMinDepth 7 -setMaxDepth 30 -doCounts 1 \
                -GL 1 -doSaf 1
done
```

and then compute the overall folded SFS by specifying the `-fold 1` option as part of the `realSFS` command:
```
for POP in JIGA PANY MBNS MAQU
do
        echo $POP
        $NGS/angsd/misc/realSFS Results/$POP.folded.saf.idx -fold 1 > Results/$POP.folded.sfs
done
```


---------------------------------------

**VERY OPTIONAL** (which means you should ignore this)

It is sometimes convenient to generate bootstrapped replicates of the SFS, by sampling with replacements genomic segments.
This could be used for instance to get confidence intervals when using the SFS for demographic inferences.
This can be achieved in ANGSD using:
```
$NGS/angsd/misc/realSFS Results/JIGA.saf.idx -bootstrap 10  2> /dev/null > Results/JIGA.boots.sfs
cat Results/JIGA.boots.sfs
```
This command may take some time.
The output file has one line for each boostrapped replicate.





---------------------------------------

Secondly, we need to estimate a **multi-dimensional SFS**, for instance the joint SFS between 2 populations (2D).
This can be used for making inferences on their divergence process (time, migration rate and so on).
However, here we are interested in estimating the 2D-SFS as prior information for our FST/PBS.

An important issue when doing this is to be sure that we are comparing the exactly same corresponding sites between populations.
ANGSD does that automatically and considers only a set of overlapping sites.

We are performing PBS assuming JIGA being the targeted population, and MAQU and MBNS as reference populations.
All 2D-SFS between such populations and MAQU are computed with:
```
POP2=JIGA
for POP in MAQU MBNS
do
        echo $POP
        $NGS/angsd/misc/realSFS Results/$POP.saf.idx Results/$POP2.saf.idx > Results/$POP.$POP2.sfs
done

# we also need the comparison between MAQU and MBNS 
$NGS/angsd/misc/realSFS Results/MAQU.saf.idx Results/MBNS.saf.idx > Results/MAQU.MBNS.sfs
```

The output file is a flatten matrix, where each value is the count of sites with the corresponding joint frequency ordered as [0,0] [0,1] and so on.
```
less -S Results/JIGA.MAQU.sfs
```
You can plot it, but you need to define how many samples (individuals) you have per population.
```
Rscript $DIR/Scripts/plot2DSFS.R Results/JIGA.MAQU.sfs 10 10
evince Results/JIGA.MAQU.sfs.pdf
```

You can even estimate SFS with higher order of magnitude.
This command may take some time and you should skip it if not interested.
```
# $NGS/angsd/misc/realSFS Results/JIGA.saf.idx Results/MBNS.saf.idx Results/MAQU.saf.idx > Results/JIGA.MBNS.MAQU.sfs
```

------------------------------------

#### 2. Population genetic differentiation

Here we are going to calculate **allele frequency differentiation** using the FST and PBS (population branch statistic) metric.
Again, we can achieve this by avoid genotype calling using ANGSD.
From the sample allele frequencies likelihoods (.saf files) we can estimate PBS using the following pipeline.

Note that here we use the previously calculated SFS as prior information.
Also, JIGA is our target population, while MAQU and MBNS are reference populations.
If not already done, you should calculate .saf.idx files for each population, as explained in the section above.

The 2D-SFS will be used as prior information for the joint allele frequency probabilities at each site.
From these probabilities we will calculate the FST and population branch statistic (PBS) using JIGA as target population and MAQU and MBNS as reference populations.
Our goal is to detect selection in JIGA in terms of allele frequency differentiation compared to MAQY and MBNS. 

**Question**
Based on the knowledge from the practicals of day 3, what pattern of selection (PBS) do you expect in JIGA? And would you expect any pronounced differentiation between MAQU and MBNS?



Specifically, we are computing a slinding windows scan, with windows of 10kbp and a step of 1kbp.
This can be achieved using the following commands.

1) This command will compute per-site FST indexes (please note the order of files). The `-whichFst 1` option is preferred Fst estimator for small sample sizes, as it is the case for our test dataset.
```
$NGS/angsd/misc/realSFS fst index Results/MAQU.saf.idx Results/MBNS.saf.idx Results/JIGA.saf.idx -sfs Results/MAQU.MBNS.sfs -sfs Results/JIGA.MAQU.sfs -sfs Results/JIGA.MBNS.sfs -fstout Results/JIGA.pbs -whichFst 1
```
and you can have a look at their values:
```
$NGS/angsd/misc/realSFS fst print Results/JIGA.pbs.fst.idx | less -S
```
where columns are: chromosome, position, (a), (a+b) values for the three FST comparisons, where FST is defined as a/(a+b).
Note that FST on multiple SNPs is calculated as sum(a)/sum(a+b).


2) The next command will perform a sliding-window analysis:
```
$NGS/angsd/misc/realSFS fst stats2 Results/JIGA.pbs.fst.idx -win 10000 -step 1000 > Results/JIGA.pbs.fst.txt
```

Have a look at the output file:
```
less -S Results/JIGA.pbs.fst.txt
```
The header is:
```
region  chr     midPos  Nsites  Fst01   Fst02   Fst12   PBS0    PBS1    PBS2
```
Where are interested in the column `PBS2` which gives the PBS values assuming our population (coded here as 2) being the target population.
Note that negative PBS and FST values are equivalent to 0.

We are also provided with the individual FST values.
You can see that high values of PBS2 are indeed associated with high values of both Fst02 and Fst12 but not Fst01.
We can plot the results along with the gene annotation.
```
Rscript $DIR/Scripts/plotPBS.R Results/MAQU.pbs.fst.txt Results/MAQU.pbs.pdf
```

It will also print out the maximum PBS value observed as this value will be used in the next part.
This script will also plot the PBS variation in JIGA as a control comparison.
```
evince Results/MAQU.pbs.pdf
```

**QUESTION**
In which part of our test sequence does JIGA display signatures of selection?



**EXERCISE**

Calculate PBS assuming PANY as target population.





-------------------------

#### 3. Nucleotide diversity

We are also interested in assessing whether an increase in allele frequency differentiation is also associated with a change of **nucleotide diversity** in JIGA.
Again, we can achieve this using ANGSD by estimating levels of diversity without relying on called genotypes.

The procedure is similar to what done for PBS, and the SFS is again used as a prior to compute allele frequencies probabilities.
From these quantities, expectations of various diversity indexes are compute.
This can be achieved using the following pipeline.

First we compute the allele frequency posterior probabilities and associated statistics (-doThetas) using the SFS as prior information (-pest)
```
POP=JIGA
$NGS/angsd/angsd -b $DIR/$POP'_bams.txt' -ref $REF -anc $ANC -out Results/$POP \
	-uniqueOnly 1 -remove_bads 1 -only_proper_pairs 1 -trim 0 -C 50 \
	-minMapQ 20 -minQ 20 -minInd 5 -setMinDepth 7 -setMaxDepth 30 -doCounts 1 \
	-GL 1 -doSaf 1 \
	-doThetas 1 -pest Results/$POP.sfs
```

Then we need to index these files and perform a sliding windows analysis using a window length of 10kbp and a step size of 1kbp.
```
POP=JIGA
# estimate for the whole region
$NGS/angsd/misc/thetaStat do_stat Results/$POP.thetas.idx
# perform a sliding-window analysis
$NGS/angsd/misc/thetaStat do_stat Results/$POP.thetas.idx -win 10000 -step 1000 -outnames Results/$POP.thetas.windows
```

Look at the results:
```
cat Results/JIGA.thetas.idx.pestPG
less -S Results/JIGA.thetas.windows.pestPG
```

The output contains many different columns: 

`#(indexStart,indexStop)(firstPos_withData,lastPos_withData)(WinStart,WinStop)   Chr     WinCenter       tW      tP      tF      tH      tL      Tajima  fuf     fud     fayh    zeng    nSites`

but we will only focus on a few here:

`Chr, WinCenter, tW, tP, Tajima and nSites`

`Chr` and `WinCenter` provide the chromosome and basepair coordinates for each window and `nSites` tells us how many genotyped sites (variant and invariant) are present in each 10kb. `tW` and `tP` are estimates of theta, namely Watterson's theta and pairwise theta. `tP` can be used to estimate the window-based pairwise nucleotide diversity (π), when we divide `tP` by the number of sites within the corresponding window (`-nSites`). 
Furthermore, the output also contains multiple neutrality statistics. As we used an outgroup to polarise the spectrum, we can theoretically look at all of these. When you only have a folded spectrum, you can't correctly estimate many of these neutrality statistics, such as Fay's H (`fayH`). However, Tajima's D (`Tajima`) can be estimated using the folded or unfolded spectrum.   

**Question**
Do regions with increased PBS values in JIGA (PBS02) correspond to regions with low Tajima's D and reduced nucleotide diversity? If so, this would be additional evidence for positive selection in JIGA. 



**EXERCISE**
Estimate the nucleotide diversity for MAQU. How do pattern of nucleotide diversity differ between MAQU and JIGA? 

------------------------


