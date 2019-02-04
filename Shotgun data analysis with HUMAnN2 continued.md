
# Shotgun analysis with HUMAnN2 - continued!

So, between last week and today, three output tables appeared for each sample - a gene family abundance table, a pathway abundance table, and a pathway coverage table. We can use Cyberduck to see these.

## Table manipulation

Before we can analyze the influence of our variables of interest, we have to manipulate the tables a little. We will need to:

1. Combine all of the tables of each type into a single gene family abundance table, pathway abundance table, and pathway coverage table.
2. Regroup the gene families by KEGG Orthogroups
3. Normalize the tables in order to account for uneven sampling depth.
4. Split the table into a stratified table and an unstratified table.
5. Convert the table to biom format.

### Combine the tables

Log into Quest.

-----
```
ssh -X <netid>@quest.it.northwestern.edu
```
-----

Start an interactive job

-----
```
msub -I -l nodes=1:ppn=2 -l walltime=02:00:00 -q short -A e30740
```
-----

Join the tables using the humann2_join_tables command. This command will take multiple tables will a single sample and give us a single table with multiple samples. We will do this for each table type.

-----
```
singularity exec -B /projects/e30740 /projects/e30740/biobakery_diamondv0822.simg humann2_join_tables --input /projects/e30740/<yourfolder>/humann2 --output /projects/e30740/<yourfolder>/humann2/allgenefamilies.tsv --file_name genefamilies

singularity exec -B /projects/e30740 /projects/e30740/biobakery_diamondv0822.simg humann2_join_tables --input /projects/e30740/<yourfolder>/humann2 --output /projects/e30740/<yourfolder>/humann2/allpathabundance.tsv --file_name pathabundance

singularity exec -B /projects/e30740 /projects/e30740/biobakery_diamondv0822.simg humann2_join_tables --input /projects/e30740/<yourfolder>/humann2 --output /projects/e30740/<yourfolder>/humann2/allpathcoverage.tsv --file_name pathcoverage
```
-----

Note, you can combine samples from different sequencing runs, projects, etc. As long as you have analyzed them using the same reference databases in HUMAnN2, they can be combined!

### Regroup the gene family abundance table by KEGG Orthogroups

For complex microbial communities, the gene family abundance table can include a very large number of features. In order to make analysis of it a little more manageable, and to make it more biologically meaningful, we likely want to collapse the gene families down into functional categories. There are many options for this in HUMAnN2 (see [here](https://bitbucket.org/biobakery/humann2/wiki/Home#markdown-header-humann2_regroup_table)), including MetaCyc reactions, Pfam domains, KEGG Orthogroups, and GO categories. Your choice will depend on your question! For the purposes of this tutorial, we'll use KEGG Orthogroups because of it's broad coverage.

Regroup the gene families using the humann2_regroup_table command.

-----
```
singularity exec -B /projects/e30740 /projects/e30740/biobakery_diamondv0822.simg humann2_regroup_table --input /projects/e30740/<yourfolder>/humann2/allgenefamilies.tsv --custom /projects/e30740/humann2_ref_data/utility_mapping/utility_mapping/map_ko_uniref50.txt.gz --output /projects/e30740/<yourfolder>/humann2/allgenefamilies_ko.tsv
```
-----

### Normalize the tables

The raw HUMAnN2 output is in reads per kilobase (RPKs), which takes into account gene length but not sequencing depth. In order to correct for uneven sequencing depth, we need to renormalize our abundance and coverage tables. Sometimes, you want to use the RPK output (ie, for strain profiling), but most of the time we will want tables that are normalized either by relative abundance or copies per million. For our downstream analysis, we will use relative abundance for the pathway tables and copies per million for the gene family table (copies per million is more meaningful when a table has a lot of features).

Normalize the tables using the humann2_renorm_table command.

-----
```
singularity exec -B /projects/e30740 /projects/e30740/biobakery_diamondv0822.simg humann2_renorm_table --input /projects/e30740/<yourfolder>/humann2/allgenefamilies_ko.tsv --output /projects/e30740/<yourfolder>/humann2/allgenefamilies_ko_cpm.tsv --units cpm

singularity exec -B /projects/e30740 /projects/e30740/biobakery_diamondv0822.simg humann2_renorm_table --input /projects/e30740/<yourfolder>/humann2/allpathabundance.tsv --output /projects/e30740/<yourfolder>/humann2/allpathabundance_relab.tsv --units relab

singularity exec -B /projects/e30740 /projects/e30740/biobakery_diamondv0822.simg humann2_renorm_table --input /projects/e30740/<yourfolder>/humann2/allpathcoverage.tsv --output /projects/e30740/<yourfolder>/humann2/allpathcoverage_relab.tsv --units relab
```
-----

### Create stratified and unstratified tables

When HUMAnN2 outputs a table, it lists both the total abundance/coverage of each pathway and the abundance/coverage of each pathway stratified or subdivided by each known organism and unknown organisms. We need to separate out these two types of data.

Split the tables into the stratified and unstratified tables using the humann2_split_stratified_table command.

-----
```
singularity exec -B /projects/e30740 /projects/e30740/biobakery_diamondv0822.simg humann2_split_stratified_table --input /projects/e30740/<yourfolder>/humann2/allgenefamilies_ko_cpm.tsv --output /projects/e30740/<yourfolder>/humann2/

singularity exec -B /projects/e30740 /projects/e30740/biobakery_diamondv0822.simg humann2_split_stratified_table --input /projects/e30740/<yourfolder>/humann2/allpathabundance_relab.tsv --output /projects/e30740/<yourfolder>/humann2/

singularity exec -B /projects/e30740 /projects/e30740/biobakery_diamondv0822.simg humann2_split_stratified_table --input /projects/e30740/<yourfolder>/humann2/allpathcoverage_relab.tsv --output /projects/e30740/<yourfolder>/humann2/
```
-----

Note, while the community-level total abundance of gene families will equal the sum of the gene family abundance from all known and unknown bacteria, that is not true for pathway abundance or pathway coverage. That is because partial reactions may be present in multiple species that, when looking at the community as a whole, sum to complete reactions. A more detailed explanation can be found [here](https://bitbucket.org/biobakery/humann2/wiki/Home#markdown-header-output-files).

### Convert the table to biom

Finally! In order to import things into QIIME2, we need to convert the tables to biom format. This makes it easy to import into QIIME2 as a feature table. Which I will certainly mistakenly refer to as an OTU-table at some point today (because I'm still stuck in QIIME1 land).

To do this we first need to make python available to ourselves

-----
```
module load python
```
-----

Then use the biom utility to convert the .tsv to a .biom, making sure we specify the correct table type. We will only convert the stratified table, as that is what is generally more useful for downstream analysis (because it incorporates taxonomy), and only worry about the gene family and pathway abundance tables for now (because they make the prettiest graphs).

-----
```
biom convert -i /projects/e30740/<yourfolder>/humann2/allgenefamilies_ko_cpm_stratified.tsv -o /projects/e30740/<yourfolder>/humann2/allgenefamilies_ko_cpm_stratified.biom --table-type "Gene table" --to-hdf5

biom convert -i /projects/e30740/<yourfolder>/humann2/allpathabundance_relab_stratified.tsv -o /projects/e30740/<yourfolder>/humann2/allpathabundance_relab_stratified.biom --table-type "Pathway table" --to-hdf5
```
-----

Then we need to remove python so it doesn't influence the next things we do.

-----
```
module purge python
```
-----

### Other things HUMAnN2 can do

HUMAnN2 can do some basic stats using the humann2_associate command. This command associates your metadata with your HUMAnN2 output tables and compare groups using a Spearman rank correlation or a Kruskal-Wallis test. These are not terribly sophisticated statistical tests for this kind of multi-dimensional data and should not be used as your final statistical analysis. But they are good for giving you a sense of what is happening in your data.

You can also do basic visualizations using the humann2_barplot command. Again, these are mostly helpful for giving you a basic sense of what your data looks like.

The [HUMAnN2 tutorial](https://bitbucket.org/biobakery/biobakery/wiki/humann2) has more information on using these commands.


## Visualization in QIIME2

Because QIIME2 does such a great job wrapping multiple tools, we are going to use it to construct PCoA plots to look visualize our data. We can do this by building a distance matrix and then constructing a PCoA plot. Because the HUMAnN2 output tables do not have associated phylogenetic data in the strictest sense, we will use a Bray-Curtis dissimilarity matrix.

First, we will need to turn our biom table into a QIIME2 feature table.

-----
```
singularity exec -B /projects/e30740 /projects/e30740/qiime2-core2018-8.simg qiime tools import --input-path /projects/e30740/<yourfolder>/humann2/allgenefamilies_ko_cpm_stratified.biom --type 'FeatureTable[RelativeFrequency]' --input-format BIOMV210Format --output-path /projects/e30740/<yourfolder>/humann2/allgenefamilies_ko_cpm_stratified.qza

singularity exec -B /projects/e30740 /projects/e30740/qiime2-core2018-8.simg qiime tools import --input-path /projects/e30740/<yourfolder>/humann2/allpathabundance_relab_stratified.biom --type 'FeatureTable[RelativeFrequency]' --input-format BIOMV210Format --output-path /projects/e30740/<yourfolder>/humann2/allpathabundance_relab_stratified.qza
```
-----

Compute beta diversity.

-----
```
singularity exec -B /projects/e30740 /projects/e30740/qiime2-core2018-8.simg qiime diversity beta --i-table /projects/e30740/<yourfolder>/humann2/allgenefamilies_ko_cpm_stratified.qza --p-metric braycurtis --output-dir /projects/e30740/<yourfolder>/humann2/genefamilies_betadiv

singularity exec -B /projects/e30740 /projects/e30740/qiime2-core2018-8.simg qiime diversity beta --i-table /projects/e30740/<yourfolder>/humann2/allpathabundance_relab_stratified.qza --p-metric braycurtis --output-dir /projects/e30740/<yourfolder>/humann2/pathabundance_betadiv
```
-----

Compute the PCoA coordinates.

-----
```
singularity exec -B /projects/e30740 /projects/e30740/qiime2-core2018-8.simg qiime diversity pcoa --i-distance-matrix /projects/e30740/<yourfolder>/humann2/genefamilies_betadiv/distance_matrix.qza --output-dir /projects/e30740/<yourfolder>/humann2/genefamilies_betadiv/pcoa

singularity exec -B /projects/e30740 /projects/e30740/qiime2-core2018-8.simg qiime diversity pcoa --i-distance-matrix /projects/e30740/<yourfolder>/humann2/pathabundance_betadiv/distance_matrix.qza --output-dir /projects/e30740/<yourfolder>/humann2/pathabundance_betadiv/pcoa
```
-----

Use emperor to plot the PCoA.

-----
```
singularity exec -B /projects/e30740 /projects/e30740/qiime2-core2018-8.simg qiime emperor plot --i-pcoa /projects/e30740/<yourfolder>/humann2/pathabundance_betadiv/pcoa/pcoa.qza --m-metadata-file /projects/e30740/humann2_tutorial_seqs/shotgun_metadata_abundance.txt --output-dir /projects/e30740/<yourfolder>/humann2/pathabundance_betadiv/plot

singularity exec -B /projects/e30740 /projects/e30740/qiime2-core2018-8.simg qiime emperor plot --i-pcoa /projects/e30740/<yourfolder>/humann2/genefamilies_betadiv/pcoa/pcoa.qza --m-metadata-file /projects/e30740/humann2_tutorial_seqs/shotgun_metadata_cpm.txt --output-dir /projects/e30740/<yourfolder>/humann2/genefamilies_betadiv/plot
```
-----

Note, the old version of QIIME had a single script for this - beta_diversity_through_plots.py. New QIIME (QIIME2) has a similar script, but requires you to include a sampling depth. This will not work for us, as we are starting with a relative abundance table that already corrects for uneven sampling depth.

## PERMANOVAs in R

Now! The moment we've all been waiting for! R!

So, close out of Quest by typing exit, hitting enter, typing exit, and hitting enter (you are exiting your interactive job and then exiting Quest).

Use CyberDuck to download the allgenefamilies_ko_cpm.tsv and allpathabundance_relab.tsv files.

Open R Studio.

First, a brief tour of R Studio!

For microbiome related statistics, we are going to use the *vegan* package in R. You can find information about it [here](https://cran.r-project.org/web/packages/vegan/index.html). Briefly, *vegan* is a package developed to handle statistics used frequently in community ecology.

We are also going to use the *data.table* library, which is a useful package for manipulating large datasets. More information can be found [here](https://cran.r-project.org/web/packages/data.table/). Similar functions exist in the *dplyr* package (part of the [tidyverse](https://www.tidyverse.org/)). *dplyr* has a slightly more intuitive syntax and *data.table* is faster for really large datasets. Either works well for what we are doing in this tutorial.

So, before we do anything, lets install those packages and load them.

-----
```
install.packages("data.table")

install.packages("vegan")

library(data.table)

library(vegan)
```
-----

Ok, now that we are all semi-oriented and have installed the necessary libraries, let's start by importing the data. We will import the normalized .tsv files from HUMAnN2. You can export the Bray-Curtis dissimilarity matrix you just made in QIIME2 as a .tsv and then import that into R, skipping some steps. But! It's helpful to know some additional functions in the *vegan* package.

First, we will set our working directory.

-----
```
setwd("/Users/elizabethmallott/Dropbox/Teaching/Anth_490_Microbiome_Analysis_Winter_2019/Humann2Day2")
```
-----

Then import the tables.

-----
```
genefamilies = fread("allgenefamilies_ko_cpm_stratified.tsv", header = T)
pathabundance = fread("allpathabundance_relab_stratified.tsv", header = T)
```
-----

Then we will transpose the tables. For smaller tables, it's possible to do this in Excel (or another spreadsheet program), but the gene family tables are often too large to open in average spreadsheet software.

-----
```
genefamilies_t1 = transpose(genefamilies) #transpose the table
names_g = genefamilies_t1[1, ] #extract the first row
setnames(genefamilies_t1,unlist(names_g)) #set the column names equal to the first row
genefamilies_t2 = genefamilies_t1[-1, ] #remove the first row
genefamilies_t = as.data.frame(genefamilies_t2) #tell R it's a dataframe
genefamilies_t[,1:52878] = lapply(genefamilies_t[,1:52878],as.numeric) #remind R that everything should be a number

pathabundance_t1 = transpose(pathabundance)
names_p = pathabundance_t1[1, ]
setnames(pathabundance_t1,unlist(names_p))
pathabundance_t2 = pathabundance_t1[-1, ]
pathabundance_t = as.data.frame(pathabundance_t2)
pathabundance_t[,1:2758] = lapply(pathabundance_t[,1:2758],as.numeric)
```
-----

Now we create the Bray-Curtis matrices.

-----
```
bray_genefamilies = vegdist(genefamilies_t[,1:52878])
bray_pathabundance = vegdist(pathabundance_t[,1:2758])
```
-----

Note: if you are importing a OTU or feature table of 16S sequences from QIIME2, you can also calculate the relative abundance of each OTU/feature using the decostand command.

In order to run any statistical analyses, we will need to import the associated metadata. Using CyberDuck, download the shotgun_metadata.tsv file from the humann2_tutorial_seqs folder. You will normally want to make sure the order of samples in your metadata file are sorted the same way as the samples in your pathway and gene family abundance tables. Luckily for us, our samples will by default be in the same numerical order in both files already.

Import the metadata into R.

-----
```
metadata = fread("shotgun_metadata.txt", header = T)
```
-----

Finally! We can run some PERMANOVAs! First we'll compare populations. This data contains three populations - a US population, a population of farmers in a remote area of Peru - Tunapuco, and individuals from the Matses - hunter-gatherers in Peru.

-----
```
adonis(formula = bray_genefamilies ~ population, data = metadata, permutations = 5000)
adonis(formula = bray_pathabundance ~ population, data = metadata, permutations = 5000)
```
-----

Then we will compare industrialized and non-industrialized humans.

-----
```
adonis(formula = bray_genefamilies ~ phyl_group_lifestyle, data = metadata, permutations = 5000)
adonis(formula = bray_pathabundance ~ phyl_group_lifestyle, data = metadata, permutations = 5000)
```
-----

Cool, right? 

All of the other statistical tests pertaining to non-phylogenetic data mentioned in the QIIME2 data analysis document also can be used with shotgun data. As you start working on your own data next week, I'm sure some of these will come up!

