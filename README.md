# Outlier Analysis Workshop

There are lots of interesting patterns that you can extract from genetic variant data. This can include patterns of linkage, balancing selection, or even inbreeding signals. One of the most common ones is to try find sites on the genome that are under selection. The following vignette will take you through the basics of genetic selection analysis. 

The project has been funded by <a href="https://ausevo.com/ECR_grants_2022/">the AES ERC Networking Grant Scheme</a> and <a href="https://genetics.org.au/">GSA</a>.

A version of this workshop that has been adapted to run directly on New Zealand eScience Infrastructure (NeSI) is <a href="https://genomicsaotearoa.github.io/Outlier_Analysis_Workshop/">available here</a>.

<h2>Schedule</h2>

**Day 1**<br>
  9:00am Introduction Slides<br>
  9:30am Download data<br>
  10:00am Morning Tea<br>
  10:15am PCAdapt<br>
  12:00pm Lunch<br>
  1:00pm VCFtools<br>
  2:00 Afternoon tea<br>
  2:15pm VCFtool continued and setup for Bayescan and Baypass<br>
  
**Day 2**<br>
  9:00am Bayescan<br>
  10:00am Morning Tea<br>
  10:15am Bayescan continued & Bayepass<br>
  12:00pm Lunch <br>
  1:00pm Baypass<br>
  2:00pm Afternoon tea<br>
  2:15pm Compiling results, group discussiona and metanalysis contribution  

<h2>A brief introduction to Genetic Outlier and Association Analysis</h2>

When we look through a genome to try find loci that are under divergent selection, we often conduct what is called outlier or association analyses. **Outlier analysis** requires just knowledge of the genetics of your samples (plus sample metadata, for example population groupings), and tries to find loci that behave very differently from the underlying patterns across the genome (with the assumption being that the rest of the genome represents patterns of neutral genetic diveristy)<sup>A</sup>. Meanwhile **association analysis** require some sort of covariate data, and tests whether there are any genetic variants statistically associated with this new data (you may have heard the term [GWAS](https://www.genome.gov/genetics-glossary/Genome-Wide-Association-Studies)). This covariate data can come in the form of phenotype data (e.g. morphology, disease status, physiology measures), or could be spatial (e.g. environmental, climate). Association tests look for sites in the genome where the precense or absence of a variant is highly correlated with the values in the co-variate data, usually through some regression type analysis.

Throughout this vignette I will refer to outlier and association analysis collectively as selection analysis. There are a lot of programs that exist currently. You can find a very long (though not exhaustive) list of them [here](https://bioinformaticshome.com/tools/gwas/gwas.html). 

This vignette still start by covering some very simple outlier analyses:
<ul>
<li><a href="https://bcm-uga.github.io/pcadapt/articles/pcadapt.html"><b>PCAdapt</b></a>, a program that can depect genetic marker outliers without having population<sup>B</sup> designations, using a Principle Component Analysis (PCA) approach.</li>
<li><a href="GBIF"><b>F<sub>ST</sub></b></a> outlier analysis, an approach that uses pairwise comparisons between two populations and the fixation index metric to assess each genetic marker.</li>
</ul>

Next we will conduct some more advanced outlier analysis:
<ul>
<li><a href="https://bcm-uga.github.io/pcadapt/articles/pcadapt.html"><b>Bayescan</b></a>, a program.</li>
<li><a href="GBIF"><b>Baypass</b></a>, a program that elaborates on the bayenv model (another popular association analysis program).</li>
</ul>

We'll cover the pre-processing of program specific input files, how to run the programs, how to visualise the output and also in some cases we'll need to take extra steps to map the genetic markers of interest back to the SNP data.  

> :beginner: **Reduced representation verses whole genome sequencing**
>
> Completing outlier analysis is possible and often done on reduced representation data. It is important to remember how your genome coverage (the number of genome variant sites / the genome length <sup>C</sup>) will affect your results and interpretation. Often with WGS data, you will see well resolved 'peaks' with a fairly smooth curve of points leading up to it either side. From this we often infer that the highest point is the genetic variant of interest, and the other sites either side of that exhibit signals of selection because they reside close to, and thus are linked, to the variant of interest. However, consider that even in WGS data, unless we have every single genetic variant represented (which may not be the case, depending on our variant calling and filtering parameters) it is possible that the genetic variant of interest that we have identified is not the main one, but is simply another neighbouring linked SNP to one that is not represented in the data. This problem becomes even more relevant with reduced representation sequencing (RRS), for which the genome coverage may be extremely patchy<sup>C</sup>. Thus with all outlier analysis, but especially so for those using RRS data, remember that your flagged outliers are not exhaustive, and may themselves only be liked to the variant that is truely under selection.

 
![ScreenShot](https://els-jbs-prod-cdn.jbs.elsevierhealth.com/cms/asset/4be56b5b-8593-4116-ab9a-ec7b9e3c9a05/gr1.jpg)



## Define you working directory for this project, and the VCF file location:

```
mkdir /home/ubuntu/outlier_analysis
DIR=/home/ubuntu/outlier_analysis
cd $DIR
```

Our data tree will look like:

> outlier_analysis/ <br>
> ├── analysis <br>
> │   ├── bayescan <br>
> │   ├── baypass <br>
> │   ├── pcadapt <br>
> │   ├── summary <br>
> │   └── vcftools_fst <br>
> ├── data  <br>
> ├── programs  <br>
> └── workshop_material <br>


So lets set up our directories to match this

```
mkdir -p {analysis/{bayescan,baypass,pcadapt,summary,vcftools_fst},data,programs,workshop_material}
```

## Project data

The data provided in this workshop contains 5007 SNPs loci for across 39 individuals (13 individuals each from 3 different locations). There is some missingness (i.e. missing SNP calls) within this data.

There is also a metadata file, that contains the individuals unique IDs, their assigned populations, and a wingspan measurement for each individual. 

Let's grab this data from the project's git repository, place the data files into our ``data`` directory, and define the environmental variables ``VCF`` and ``METADATA`` with the locations of the genetic variant and metadata files respectively.

```
cd $DIR/workshop_material
git clone https://github.com/katarinastuart/Ev1_SelectionMetaAnalysis.git
cp $DIR/workshop_material/Ev1_SelectionMetaAnalysis/workshop_files/* $DIR/data
VCF=$DIR/data/starling_3populations.recode.vcf
METADATA=$DIR/data/starling_3populations_metadata.txt
```

> :heavy_exclamation_mark: **Working with your own data** <br> 
> <br>
> Alternatively, you can also use your own data for this workshop. If so, it is a good idea to thin your SNP dataset down to roughly 5,000 SNPs to ensure compute times are not too long. If you have more than 50 individuals you may also want to reduce this too. If you would like to do this, just place your genetic variant and metadata file in the ``data`` directory and define ``VCF`` and ``METADATA`` based on their names. <br>


Across this workshop, we will need the genetic data to be in several different formats. Let's prepare that now. First we convert the VCF to PLINK, and then to BED.

```
cd $DIR/data
module load quay.io/biocontainers/vcftools/0.1.15--he941832_2/module
module load quay.io/biocontainers/plink/1.90b6.21--hec16e2b_2/module
vcftools --vcf $VCF --plink --out starling_3populations.plink
plink --file starling_3populations.plink --make-bed --noweb --out starling_3populations
```



## PCAdapt

PCAdapt uses an ordination approach to find sites in a data set that are outliers with respect to background population structure. The PCAdapt manual is available [here](https://bcm-uga.github.io/pcadapt/articles/pcadapt.html). 

CITE: Privé, F., Luu, K., Vilhjálmsson, B. J., & Blum, M. G.B. (2020). Performing highly efficient genome scans for local adaptation with R package pcadapt version 4. Molecular Biology and Evolution.


First, let's install PCAdapt and set your working directory.

```
R

install.packages("pcadapt")
library("pcadapt")

setwd("/home/ubuntu/outlier_analysis/analysis/pcadapt/")
```

Now let's load in the data - PCAdapt uses bed file types.

```
starling_bed <- "/home/ubuntu/outlier_analysis/data/starling_3populations.bed"
starlings_pcadapt <- read.pcadapt(starling_bed, type = "bed")
```

Produce K plot

```
starlings_pcadapt_kplot <- pcadapt(input = starlings_pcadapt, K = 20)
pdf("pcadapt_starlings_kplot.pdf")
plot(starlings_pcadapt_kplot, option = "screeplot")
dev.off()
```

<img src="/images/pcadapt_kplot.PNG" alt="k plot" width="400"/>

K value of 3 is most appropriate, as this is the value of K after which the curve starts to flatten out more.

```
starlings_pcadapt_pca <- pcadapt(starlings_pcadapt, K = 3)
summary(starlings_pcadapt_pca)
```

> :heavy_check_mark: **output** <br>
> &nbsp; &nbsp; &nbsp; &nbsp; Length Class  Mode <br>
> scores &nbsp; 117  -none- numeric <br>
> singular.values &nbsp;     3  -none- numeric <br>
> loadings &nbsp;        15021  -none- numeric <br>
> zscores &nbsp;         15021  -none- numeric <br>
> af &nbsp;               5007  -none- numeric <br>
> maf &nbsp;              5007  -none- numeric <br>
> chi2.stat &nbsp;       5007  -none- numeric <br>
> stat &nbsp;             5007  -none- numeric <br>
> gif &nbsp;                 1  -none- numeric <br>
> pvalues &nbsp;          5007  -none- numeric <br>
> pass &nbsp;             4610  -none- numeric 

Investigate axis projections:

```
poplist.names <- c(rep("Lemon", 13),rep("Warrnambool", 13),rep("Nowra", 13))
print(poplist.names)

pdf("pcadapt_starlings_projection1v2.pdf")
plot(starlings_pcadapt_kplot, option = "scores", i = 1, j = 2, pop = poplist.names)
dev.off()

pdf("pcadapt_starlings_projection5v7.pdf")
plot(starlings_pcadapt_kplot, option = "scores", i = 5, j = 7, pop = poplist.names)
dev.off()
```

Ignore the warning: 

> :heavy_exclamation_mark: Use of `df$Pop` is discouraged. Use `Pop` instead.

<img src="/images/pcadapt_proj1.PNG" alt="projection axis1 axis2" width="300"/> <img src="/images/pcadapt_proj2.PNG" alt="projection axis6 axis7" width="300"/>

Investigate manhattan and Q-Qplot:

> :beginner: **Manhattan plots** are a way to visualise the GWAS (genome-wide association study) p-values (or other statistical values) at each SNP locus along the genome

> :beginner: **Q-Qplots plots** are just a quick way to visually check if your residuals are normally distributed. Check out more information [here](https://data.library.virginia.edu/understanding-q-q-plots/).

```
pdf("pcadapt_starlings_manhattan.pdf")
plot(starlings_pcadapt_pca, option = "manhattan")
dev.off()

pdf("pcadapt_starlings_qqplot.pdf")
plot(starlings_pcadapt_pca, option = "qqplot")
dev.off()
```

<img src="/images/pcadapt_manhattan.PNG" alt="Manhattan" width="400"/> <img src="/images/pcadapt_qq.PNG" alt="Q-Qplot" width="400"/>

Plotting and correcting the pvalues

```
starling_pcadapt_pvalues <- as.data.frame(starlings_pcadapt_pca$pvalues)

library("ggplot2")

pdf("pcadapt_starlings_pvalues.pdf")
hist(starlings_pcadapt_pca$pvalues, xlab = "p-values", main = NULL, breaks = 50, col = "orange")
dev.off()
```

<img src="/images/pcadapt_pvals.PNG" alt="pvals" width="450"/>

```
starlings_pcadapt_padj <- p.adjust(starlings_pcadapt_pca$pvalues,method="bonferroni")
alpha <- 0.1
outliers <- which(starlings_pcadapt_padj < alpha)
length(outliers)

write.table(outliers, file="starlings_pcadapt_outliers.txt") 

length(outliers)
```

> :heavy_check_mark: **Output** <br>
> &emsp;
> [1] 3

After this, we will be jumping out of R and back into the command line by using the command: 

```
q()
```

Mapping Outliers: PCAdapt

finding the SNP ID of the outlier variants

```
cd $DIR/analysis
```

The first thing we will do is create list of SNPs in VCF, assign line numbers that can be used to find matching line numbers in outliers (SNP ID is lost in PCadapt & Bayescan, line numbers used as signifiers). 

We create this in the ``analysis`` folder because we will use it for more than just mapping the outlier SNPs for PCAdapt.

```
grep -v "^#" $DIR/data/starling_3populations.recode.vcf | cut -f1-3 | awk '{print $0"\t"NR}' > starling_3populations_SNPs.txt
```

Now let's jump back into the ``pcadapt`` directory to contiue working with our outliers. We grab column 2 of the outlier file using the ``AWK`` command, which contain the number of the outliers

```
cd $DIR/analysis/pcadapt
awk '{print $2}' starlings_pcadapt_outliers.txt > starlings_pcadapt_outliers_numbers.txt
```

We now make a list of outlier SNPS ID's

```
awk 'FNR==NR{a[$1];next} (($4) in a)' starlings_pcadapt_outliers_numbers.txt ../starling_3populations_SNPs.txt   | cut -f3 > pcadapt_outlierSNPIDs.txt
head pcadapt_outlierSNPIDs.txt
```
> :heavy_check_mark: **Output** <br>
> &emsp; <br>
> 230955:72:- <br>
> 238881:46:+ <br>
> 286527:46:- <br>



## VCFtools windowed Fst

The VCFTools manual is available [here](https://vcftools.sourceforge.net/man_latest.html).

CITE: Danecek P, Auton A, Abecasis G, Albers CA, Banks E, DePristo MA, Handsaker RE, Lunter G, Marth GT, Sherry ST et al. 2011 The variant call format and VCFtools. Bioinformatics 27 2156–2158. (doi:10.1093/bioinformatics/btr330)

Fst outliers will allow us to identify SNPs that behave abnormally in pairwise comparisons between populations.

The first things we need to do is use our metadata file (currently defined by the environmental variable ``METADATA``) to make three individual files containing just the list of individuals in each of the populations. We can do this by subseting our sample metadata file, using the command ``grep`` to grab lines that match each population's name, and then using ``awk`` to keep only the first column of metadta, i.e. the sample names.

```
module load quay.io/biocontainers/vcftools/0.1.15--he941832_2/module
```

```
cd $DIR/data

grep "Lemon" $METADATA | awk '{print $1}' > individuals_Lemon.txt
grep "War" $METADATA | awk '{print $1}' > individuals_War.txt
grep "Nowra" $METADATA | awk '{print $1}' > individuals_Nowra.txt
```

Now we can pick two populations to compare. Let's work with Lemon (short for Lemon Tree, QLD, AU) and War (short for Warnambool, VIC, AU), and so a SNP-based Fst comparison.

```
cd $DIR/analysis/vcftools_fst

vcftools --vcf $VCF --weir-fst-pop $DIR/data/individuals_Lemon.txt --weir-fst-pop $DIR/data/individuals_War.txt --out lemon_war

head -n 5 lemon_war.weir.fst
```
> :heavy_check_mark: **Output** <br>
> &emsp;
> CHROM   POS     WEIR_AND_COCKERHAM_FST<br>
> starling4       107735  0.160891<br>
> starling4       137462  -0.0805785<br>
> starling4       151332  0.0524489<br>
> starling4       227887  0.0569961<br>

The important column is column 5: the Weighted Fst, from [Weir and Cockerham’s 1984 publication](https://www.jstor.org/stable/2408641). This corrects for INFO NEEDED.

```
wc -l lemon_war.weir.fst 
```

> :heavy_check_mark: **Output** <br>
> &emsp;
> 5008

Notice how there are as many lines as there are SNPs in the data set, plus one for a header. It is always a good idea to check your output, and make sure everything looks as you expect it to!

Next, instead of calculating pairwise populaiton differentiation on a SNP by SNP basis, we will be usiing a sliding window approach. The ``--fst-window-size 50000`` refers to the window size of the genome (in bsae pairs) in which we are calculating one value: all SNPs within this window are used to caluclate Fst. The ``--fst-window-step`` option indicates how many base pairs the window is moving down the genome before calculating Fst for the second window, and then the third, and so on. 

> **Warning**
> &emsp;
> These sliding windows only work on ordered SNPs on the same chromosome/scaffold/contig. If you data is not set up like this (i.e. all your SNPs are on a single pseudo chromosome) then this method is not appropriate for your data, as it will be making an assumption about where the SNPs are located with respect to one another.

```
vcftools --vcf $VCF --fst-window-size 50000 --fst-window-step 10000 --weir-fst-pop $DIR/data/individuals_Lemon.txt --weir-fst-pop $DIR/data/individuals_War.txt --out lemon_war

head lemon_war.windowed.weir.fst
```
> :heavy_check_mark: **Output** <br>
> &emsp;
> CHROM   BIN_START       BIN_END N_VARIANTS      WEIGHTED_FST    MEAN_FST <br>
> starling4       60001   110000  1       0.160891        0.160891 <br>
> starling4       70001   120000  1       0.160891        0.160891 <br>
> starling4       80001   130000  1       0.160891        0.160891 <br>
> starling4       90001   140000  2       -0.00374291     0.0401563 <br>


Notice the output is different.

```
wc -l lemon_war.windowed.weir.fst 
```

> :heavy_check_mark: **Output** <br>
> &emsp;
> 10838

Notice the line count is different from the SNP-based Fst comparison; there are more lines in the sliding window based Fst comparison. This is because there are more sliding windows across the chromosome in this data set than there are SNPs. Consider which of these steps is better for your data: in low density SNP datasets, the sliding window approach might not be the best to use.

Now let's plot the Fst across the chromosome. To do this we will add line numbers on our Fst file that will be used to order the Fst measurements across the x-axis of our manhattan plot.

> :beginner: **X-axis values** in the following plot are done y using each ourlier window's line number, as they are in order along the genome. Ourlier windows are equally spces, and so line numbers are sufficient to capture the patterns along the genome. Consider that if you are plotting Fst values for SNPs (rather than windows), they may not be equally spaced along the genome and so SNP position may need to be used to make your manhattan plots.

```
awk '{print $0"\t"NR}' ./lemon_war.windowed.weir.fst  > lemon_war.windowed.weir.fst.edit

R

library("ggplot2")

setwd("/home/ubuntu/outlier_analysis/analysis/vcftools_fst")

windowed_fst <- read.table("lemon_war.windowed.weir.fst.edit", sep="\t", header=TRUE)
str(windowed_fst)

quantile(windowed_fst$WEIGHTED_FST, probs = c(.95, .99, .999))
```

> :heavy_check_mark: **Output** <br>
> &emsp;
>       95%       99%     99.9%<br>
> 0.1948850 0.3501600 0.5741306<br>


Choose the quantile threshold above which SNPs will be classified as outliers. Below, we chose 99% (i.e. the top 1% of SNP windows).

```
pdf("fst_starlings_windowed.pdf", width=10, height=5)
ggplot(windowed_fst, aes(x=X1, y=WEIGHTED_FST)) + 
geom_point() + 
theme_classic() +
geom_hline(yintercept=0.35, linetype="dashed", color = "red")
dev.off()

q()
```

<img src="/images/Fst_Windowed.PNG" alt="Windowed Fst" width="600"/>

Finally, we will generate a list of outlier SNP IDs. We do this by grabbing all of the SNPs located in the outlier windows. 

```
cd $DIR/analysis/vcftools_fst
cat lemon_war.windowed.weir.fst.edit | awk '$5>0.3501600' > lemon_war.windowed.outliers
wc -l lemon_war.windowed.outliers
```
> :heavy_check_mark: **Output** <br>
> &emsp;
> 107 lemon_war_fst.windowed.outliers

```
cut -f1-3 lemon_war.windowed.outliers > lemon_war.windowed.outliers.bed 
vcftools --vcf $VCF --bed lemon_war.windowed.outliers.bed --out fst_outliers --recode
grep -v "#" fst_outliers.recode.vcf | awk '{print $3}' > vcftoolsfst_outlierSNPIDs.txt
wc -l vcftoolsfst_outlierSNPIDs.txt
```
> :heavy_check_mark: **Output** <br>
> &emsp;
> 61

We have a total of 61 outlier SNPs locate across 107 outlier SNP windows.


## Bayescan

The VCFTools manual is available [here](https://github.com/mfoll/BayeScan).

CITE: Foll M & Gaggiotti O 2008 A Genome-Scan Method to Identify Selected Loci Appropriate for Both Dominant and Codominant Markers: A Bayesian Perspective. Genetics 180 977–993. (doi:10.1534/genetics.108.092221)

Bayescan identified outlier SNPs based on allele frequencies. More explination about alpha and such.

First, we will need to convert out VCF to the Bayescan format. To do this we will use the genetic file conversion program called [PGDspider](http://www.cmpg.unibe.ch/software/PGDSpider/). 

We also need to create a new populations metadata file, which contains individual names in column 1, and population names in column 2.

```
cd $DIR/data
cut -f1,2 $METADATA > starling_3populations_metadata_INDPOP.txt
```

Now navigate to the folder in which we will be running the bayescan analysis.

```
cd $DIR/analysis/bayescan
```

We now run PGDSpider in two steps: first we convert the VCF file to the PGD format, second from PGD format to Bayescan format. To do this we will need to create a SPID file. create a file called *VCF_PGD.spid* using the ``nano`` command. Paste in the below, replacing the location of you metadata file.

```
nano VCF_PGD.spid
```
include snapshot of SPID:

```
# VCF Parser questions 
PARSER_FORMAT=VCF 
# Only output SNPs with a phred-scaled quality of at least: 
VCF_PARSER_QUAL_QUESTION= 
# Select population definition file: 
VCF_PARSER_POP_FILE_QUESTION=/home/ubuntu/outlier_analysis/data/starling_3populations_metadata_INDPOP.txt 
# What is the ploidy of the data?
VCF_PARSER_PLOIDY_QUESTION=DIPLOID
# Do you want to include a file with population definitions?
VCF_PARSER_POP_QUESTION=true
# Output genotypes as missing if the phred-scale genotype quality is below:
VCF_PARSER_GTQUAL_QUESTION=
# Do you want to include non-polymorphic SNPs?
VCF_PARSER_MONOMORPHIC_QUESTION=false
# Only output following individuals (ind1, ind2, ind4, ...):
VCF_PARSER_IND_QUESTION=
# Only input following regions (refSeqName:start:end, multiple regions: whitespace separated):
VCF_PARSER_REGION_QUESTION=
# Output genotypes as missing if the read depth of a position for the sample is below:
VCF_PARSER_READ_QUESTION=
# Take most likely genotype if "PL" or "GL" is given in the genotype field?
VCF_PARSER_PL_QUESTION=false
# Do you want to exclude loci with only missing data?
VCF_PARSER_EXC_MISSING_LOCI_QUESTION=false

# PGD Writer questions
WRITER_FORMAT=PGD 
```

Now run the two step conversion.

```
module load quay.io/biocontainers/pgdspider/2.1.1.5--hdfd78af_1/module

PGDSpider2-cli -inputfile $VCF -inputformat VCF -outputfile starling_3populations.pgd -outputformat  PGD -spid VCF_PGD.spid 

PGDSpider2-cli -inputfile starling_3populations.pgd -inputformat PGD -outputfile starling_3populations.bs -outputformat GESTE_BAYE_SCAN
```
Let's have a quick look at what the input file looks like:

```
head starling_3populations.bs
```

> :heavy_check_mark: **Output** <br>
> [loci]=5007 <br>
> 
> [populations]=3 <br>
> 
> [pop]=1 <br>
> 1      12      2       9 3 <br>
> 2      20      2       11 9 <br>
> 3      18      2       15 3 <br>
> 4      20      2       0 20 <br>
> 5      22      2       2 20 <br>

So for each population we have a note of how many REF and ALT alleles we have at each genomic variant position.

> :beginner: **An important note about additive genetic variance**: It is important to bear in mind how the input genetic data for outlier or association models is being interpreted by the model. When dealing with many of these models (and input genotype files) the assumption is that the SNP effects are [additive](https://link.springer.com/referenceworkentry/10.1007/978-3-319-47829-6_5-1). This can be seen from, for example, the way we encode homozygous reference allele, heterozygous, and homozygous alternate allele as "0", "1", and "2" respectively in a BayPass input genofile. For the diploid organism (with two variant copies for each allele) one copy of a variant (i.e. heterozygous) is assumed to have half the effect of having two copies. However, what if the locus in question has dominance effects? This would mean the heterozygous form behaves the same as the homozygous dominant form, and it would be more appropriate to label these instead as "0", "0", "1". But with thousands, if not millions of (most likely) completely unknown variants in a dataset, how can we possibly know? The answer is we cannot. And most models will assume additive effects, because this the simplest assumption. However, by not factoring in dominance effects we could possible be missing many important functional variants, as Reynolds et al. [2021](https://www.nature.com/articles/s41588-021-00872-5) demonstrates. Genomics is full of caveats and pitfalls, which while providing new directions to explore can be a bit overwhelming. Remember, you selection analysis doesn't have to be exhaustive, just make sure it is as fit for purpose within your study design. There is so much going on in just one genome, there is no way you can analyse everything in one go. 


Now let's set Bayescan to run. Using the ``nano bayescan_starling.sl`` we will create a slurm script and submit it to run using the command ``sbatch bayescan_starling.sl``. This should take approximately 1 hr to run. Currently everything is set to default, but read the manual if you want to understand what they mean and how to refine them if needed.

```
#!/bin/bash -e

#load bayescan
module load quay.io/biocontainers/bayescan/2.0.1--h4ac6f70_6

#run bayescan. 
bayescan_2 ./starling_3populations.bs -od ./ -threads 8 -n 5000 -thin 10 -nbp 20 -pilot 5000 -burn 50000 -pr_odds 10
```
 
Identify outliers:

```
module load R/4.1.0-gimkl-2020a
R
library(ggplot2)
setwd("/nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/analysis/bayescan")
source("/opt/nesi/CS400_centos7_bdw/BayeScan/2.1-GCCcore-7.4.0/R\ functions/plot_R.r")
outliers.bayescan=plot_bayescan("starling_3population_fst.txt",FDR=0.05)
outliers.bayescan
write.table(outliers.bayescan, file="bayescan_outliers.txt")

q()
```
> :heavy_check_mark: **Output** <br>
> &emsp;
> $outliers <br>
> [1] 333  395 1367 2376 3789 <br>
> $nb_outliers <br>
> [1] 5

Mapping Outliers

```
cd $DIR/analysis/bayescan
```

Create list of SNPs in VCF, assign line numbers that can be used to find matching line numbers in outliers (SNP ID is lost in bayescan, line numbers used as signifiers).

```
grep -v "^#" ../../data/starling_3populations.recode.vcf  | cut -f1-3 | awk '{print $0"\t"NR}' > starling_3populations_SNPs.txt

awk '{print $2}' bayescan_outliers.txt > bayescan_outliers_numbers.txt
```

list of outlier SNPS, by matching column 1 of of the outliers list to the fourth column of the whole SNP list data.

```
awk 'FNR==NR{a[$1];next} (($4) in a)' bayescan_outliers_numbers.txt starling_3populations_SNPs.txt   | cut -f3 > bayescan_outlierSNPIDs.txt
```

Bayescane Log Plot, colouring the outliers in a different colour.

```
module load R/4.1.0-gimkl-2020a
R
library(ggplot2)
library(dplyr)
setwd("/nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/analysis/bayescan")

bayescan.out<- read.table("starling_3population_fst.txt", header=TRUE)
bayescan.out <- bayescan.out %>% mutate(ID = row_number())
bayescan.outiers<- read.table("bayescan_outliers_numbers.txt", header=FALSE)
outliers.plot <- filter(bayescan.out, ID %in% bayescan.outiers[["V1"]])

png("bayescan_outliers.png", width=600, height=350)
ggplot(bayescan.out, aes(x=log10.PO., y=alpha))+
geom_point(size=5,alpha=1)+xlim(-1.3,3.5)+ theme_classic(base_size = 18) + geom_vline(xintercept = 0, linetype="dashed", color = "black", size=3)+
geom_point(aes(x=log10.PO., y=alpha), data=outliers.plot, col="red", fill="red",size=5,alpha=1) + theme(axis.text=element_text(size=18), axis.title=element_text(size=22,face="bold"))
dev.off()

q()
```
<img src="/images/bayescan_outliers.png" alt="Windowed Fst" width="300"/>

## BayPass

The Baypass manual can be found [here](http://www1.montpellier.inra.fr/CBGP/software/baypass/files/BayPass_manual_2.31.pdf).

CITE: Gautier M 2015 Genome-Wide Scan for Adaptive Divergence and Association with Population-Specific Covariates. Genetics 201 1555–1579. (doi:10.1534/genetics.115.181453)

Baypass requires that the allele frequency data be on a population, not an individual basis. The genotyping data file is simply organized as a matrix with nsnp rows and 2 ∗ npop columns. The row field separator is a space. More precisely, each row corresponds to one marker and the number of columns is twice the number of populations because each pair of numbers corresponds to each allele (or read counts for PoolSeq experiment) counts in one population. 

To generate this population gene count data we will work with the PLINK file. First we have to fix the individual ID and population labels, as PLINK has pulled these directly from the VCF which has no popuation information. What we are aiming for is population in column 1, and individual ID in column 2.

```
cd $DIR/data

awk '{print $2,"\t",$1}' $METADATA > starling_3populations_metadata_POPIND.txt

cd $DIR/analysis/baypass

PLINK=$DIR/data/starling_3populations.plink.ped

#remove first 2 columns
cut -f 3- $PLINK > x.delete

paste $DIR/data/starling_3populations_metadata_POPIND.txt x.delete > starling_3populations.plink.ped
rm x.delete 

cp $DIR/data/starling_3populations.plink.map .
cp $DIR/data/starling_3populations.plink.log .
```

run the pop based allele frequency calculations

```
module load PLINK/1.09b6.16 

plink --file starling_3populations.plink --allow-extra-chr --freq counts --family --out starling_3populations
```

manipulate file so it has baypass format, numbers set for plink output file and pop number for column count

```
tail -n +2 starling_3populations.frq.strat | awk '{ $9 = $8 - $7 } 1' | awk '{print $7,$9}' | tr "\n" " " | sed 's/ /\n/6; P; D' > starling_3populations_baypass.txt
```

Now we can run Baypass by creating a slurm script ``baypass1_starling.sl``, which should run for about 5 minutes.

```
#!/bin/bash -e
#SBATCH --job-name=2023_04_14.baypass1_starling.sl
#SBATCH --account=uoa02613
#SBATCH --time=00-12:00:00
#SBATCH --mem=5GB
#SBATCH --output=%x_%j.errout
#SBATCH --mail-user=katarina.stuart@auckland.ac.nz
#SBATCH --mail-type=ALL
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --profile task

module load BayPass/2.31-intel-2022a

cd /nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/analysis/baypass

i_baypass -npop 3 -gfile ./starling_3populations_baypass.txt -outprefix starling_3populations_baypass -nthreads 4
```

Running in R to make the anapod data. First let's quickly download the utils we need.

```
cd $DIR/programs
git clone https://github.com/andbeck/BayPass.git
```
Now let's generate some simulated data, based on the parameters calculated from our genetic data.

```
module load R/4.1.0-gimkl-2020a
R
setwd("/nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/analysis/baypass")
source("/nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/programs/BayPass/baypass_utils.R")

install.packages("ape")
library("ape")

install.packages("corrplot")
library("corrplot")

omega=as.matrix(read.table("starling_3populations_baypass_mat_omega.out"))
pi.beta.coef=read.table("starling_3populations_baypass_summary_beta_params.out",h=T)$Mean
bta14.data<-geno2YN("starling_3populations_baypass.txt")
simu.bta<-simulate.baypass(omega.mat=omega, nsnp=5000, sample.size=bta14.data$NN, beta.pi=pi.beta.coef,pi.maf=0,suffix="btapods")

q()
```

We now have the simulated genetic data, we can find the XtX statistic threshold above which we will consider genetic sites an outlier. Let's create another script in ``cd $DIR/analysis/baypass`` called ``baypass2_starling.sl``.

```
#!/bin/bash -e
#SBATCH --job-name=2023_04_14.baypass2_starling.sl
#SBATCH --account=uoa02613
#SBATCH --time=00-12:00:00
#SBATCH --mem=5GB
#SBATCH --output=%x_%j.errout
#SBATCH --mail-user=katarina.stuart@auckland.ac.nz
#SBATCH --mail-type=ALL
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --profile task

module load BayPass/2.31-intel-2022a

cd /nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/analysis/baypass

i_baypass -npop 3 -gfile G.btapods  -outprefix G.btapods -nthreads 4 
```

XtX calibration; get the pod XtX theshold

```
module load R/4.1.0-gimkl-2020a
R
setwd("/nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/analysis/baypass")
source("/nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/programs/BayPass/baypass_utils.R")
library("ape")
library("corrplot")

pod.xtx=read.table("G.btapods_summary_pi_xtx.out",h=T)$M_XtX
```

We compute the 1% threshold for the simulated neutral data.

```
pod.thresh=quantile(pod.xtx,probs=0.99)
pod.thresh

q()
```

> :heavy_check_mark: **Output** <br>
> &emsp;
> 6.258372

Your values may be slightly different as the simlated data will not be identical.

Next, we filter the data for the outlier snps by identifying those above the threshold.

```
cat starling_3populations_baypass_summary_pi_xtx.out | awk '$4>6.258372 ' > baypass_outliers.txt
```

create list of SNPs in VCF, assign line numbers that can be used to find matching line numbers in outliers (SNP ID is lost in bayescan, line numbers used as signifiers).

```
grep -v "^#" $VCF  | cut -f1-3 | awk '{print $0"\t"NR}' > starling_3populations_SNPs.txt
```

List of outlier SNPS

```
awk 'FNR==NR{a[$1];next} (($4) in a)' baypass_outliers.txt starling_3populations_SNPs.txt | cut -f3 > baypass_outlierSNPIDs.txt
wc -l baypass_outlierSNPIDs.txt
```

> :heavy_check_mark: **Output** <br>
> &emsp;
> 38

Low lets find SNPs that are statistically associated with wingspan. To do this we have to go back to the metadata and compute the average wingspan of each of out populations and place them in a file.

```
module load R/4.1.0-gimkl-2020a
R
setwd("/nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/analysis/baypass")
metadata <- read.table("/nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/data/starling_3populations_metadata.txt", sep="\t", header=FALSE)
str(metadata)
pop_metadata <- aggregate(V3 ~ V2, data = metadata, mean)
write(pop_metadata[,2], "pop_mean_wingspan.txt")

q()
```

> :heavy_check_mark: **Output** <br>
> &emsp;
> 14.89805 19.63306 22.09655

Now we can submit a third and final baypass job ``baypass3_starling.sl``, which will let us know which SNPs are statistically associated with wingspan.


```
#!/bin/bash -e
#SBATCH --job-name=2023_04_14.baypass3_starling.sl
#SBATCH --account=uoa02613
#SBATCH --time=00-12:00:00
#SBATCH --mem=5GB
#SBATCH --output=%x_%j.errout
#SBATCH --mail-user=katarina.stuart@auckland.ac.nz
#SBATCH --mail-type=ALL
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --profile task

module load BayPass/2.31-intel-2022a

cd /nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/analysis/baypass

i_baypass -npop 3 -gfile starling_3populations_baypass.txt -efile pop_mean_wingspan.txt -scalecov -auxmodel -nthreads 4 -omegafile starling_3populations_baypass_mat_omega.out -outprefix starling_3populations_baypass_wing
```


Next we plot the outliers. We are chosing a <a href="https://www.statology.org/bayes-factor/">BF threshold</a> of 20 dB, which indicates "Strong evidence for alternative hypothesis".

```
module load R/4.1.0-gimkl-2020a
R

library(ggplot2)

setwd("/nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/analysis/baypass")

covaux.snp.res.mass=read.table("starling_3populations_baypass_wing_summary_betai.out",h=T)
covaux.snp.xtx.mass=read.table("starling_3populations_baypass_summary_pi_xtx.out",h=T)$M_XtX

pdf("Baypass_plots.pdf")
layout(matrix(1:3,3,1))
plot(covaux.snp.res.mass$BF.dB.,xlab="Mass",ylab="BFmc (in dB)")
abline(h=20, col="red")
plot(covaux.snp.res.mass$M_Beta,xlab="SNP",ylab=expression(beta~"coefficient"))
plot(covaux.snp.xtx.mass, xlab="SNP",ylab="XtX corrected for SMS")
dev.off()
```

<img src="/images/Baypass.PNG" alt="Baypass output" width="500"/>

Finally, lets generate the list of phenotype-associated SNP IDs. 

```
cat starling_3populations_baypass_wing_summary_betai.out | awk '$6>20' > starling_3populations_baypass_wing_BF20.txt
starling_3populations_baypass_wing_BF20.txt

wc -l starling_3populations_baypass_wing_BF20.txt
```

> :heavy_check_mark: **Output** <br>
> &emsp;
> 48

Filtering the data sets for SNPS above BFmc threshold - these are out outlier SNPs that are associated with wingspan. 

```
awk 'FNR==NR{a[$2];next} (($4) in a)' starling_3populations_baypass_wing_BF20.txt starling_3populations_SNPs.txt | cut -f3 > baypass_wingspan_outlierSNPIDs.txt

comm -12 <(sort baypass_wingspan_outlierSNPIDs.txt) <(sort baypass_outlierSNPIDs.txt) > double_outliers.txt

wc -l double_outliers.txt
```

> :heavy_check_mark: **Output** <br>
> &emsp;
> 18

## Comparing Outlier Overlap

Now we wil make an upset plot to compare the overlap of outliers detected over our different methods.

```
cd $DIR/analysis/summary
ln -s $DIR/analysis/pcadapt/pcadapt_outlierSNPIDs.txt .
ln -s $DIR/analysis/vcftools_fst/vcftoolsfst_outlierSNPIDs.txt .
ln -s $DIR/analysis/bayescan/bayescan_outlierSNPIDs.txt .
ln -s $DIR/analysis/baypass/baypass_outlierSNPIDs.txt .
ln -s $DIR/analysis/baypass/baypass_wingspan_outlierSNPIDs.txt .
```

Now we have a copy of all the SNP IDs for each of out outlier analysis, let's use the R package <a href="https://cran.r-project.org/web/packages/UpSetR/vignettes/basic.usage.html">UpSetR</a> to plot the overlap.

```
module load R/4.1.0-gimkl-2020a
R
setwd("/nesi/nobackup/uoa02613/kstuart_projects/outlier_analysis/analysis/summary")

pcadapt<-scan("pcadapt_outlierSNPIDs.txt", what = "", quiet=TRUE)
vcftools<-scan("vcftoolsfst_outlierSNPIDs.txt", what = "", quiet=TRUE)
bayescan<-scan("bayescan_outlierSNPIDs.txt", what = "", quiet=TRUE)
baypass<-scan("baypass_outlierSNPIDs.txt", what = "", quiet=TRUE)
baypass_wing<-scan("baypass_wingspan_outlierSNPIDs.txt", what = "", quiet=TRUE)  

all_outliers <- list(PCAdapt = pcadapt, VCFtools = vcftools, Bayescan = bayescan, Baypass = baypass, BaypassWing = baypass_wing)

install.packages("UpSetR")
library(UpSetR)

pdf("All_outliers_upsetplot.pdf")
upset(fromList(all_outliers), order.by = "freq", empty.intersections = "on", point.size = 3.5, line.size = 2, mainbar.y.label = "Outlier Count", sets.x.label = "Total Outliers", text.scale = c(1.3, 1.3, 1, 1, 2, 1.3), number.angles = 30, nintersects = 11 ) 
dev.off() 
```

<img src="/images/outliers_upsetplot.PNG" alt="upset plot of outlier overlaps" width="500"/>

Let's have a discussion about the overlap between these five outlier groups. <p>

And if you want to get really fancy you may event want to plot your variants at their location around your genome in a <a href="https://github.com/katarinastuart/Sv3_StarlingGenome">circle plot</a>!

## Workshop End discussion
  
A brief period of group discussion on one of the days about research question framing + grant integration
  
## Outlier Analysis Metanalysis

This workshop was concieved as part of a larger project. The goal is to compile as many genomics data set with identified outliers as possible. While identifying outliers is an interesting and often necessary component of singular genomics projects, there is also a lot to be gained from looking at patterns across neutral versus outlier genetic variants across many different projects and taxa. <p>

One of the goals of this project is to compile a collection of genetic data sets information. Most of these will come from prepublished work, but attendees of this workshop may opt in their data sets should they want to have their data invovled in this project. 

**Ideally for the metanalysis we need:**<br>
VCF file with all genetic variants (SNPS) <br>
List of which variants are outliers, and what type of outliers these are <br>
OPTIONAL but PREFERED: Reference genome that has been annotated and well scaffolded <br>


## Funding 
<p align="center">

![ScreenShot](https://storage.corsizio.com/uploads/5cea29e798d9a757e03dba1c/events/6141a39502de2a7ff2f3f120/photo-_3UnQXVrb.jpg)

</p>

Thank you to the <a href="https://ausevo.com/ECR_grants_2022/">AES ERC Networking Grant Scheme</a> and <a href="https://genetics.org.au/">GSA</a> for funding this project.

## Footnotes
<sup>A</sup> it is very important then to account for any population substructure. There are many different ways to approach this: refer to introduction slides for some guidance.<br>
<sup>B</sup> I will say population for simplicity throughout this vignette. However, equally we can test for differences between sample sites, subpopulations, and other types of groupings. What counts as one 'group' of organisms will be dependent on your study system or study question. <br>
<sup>C</sup> You may also want to consider linkage blocks.
