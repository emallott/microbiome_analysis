
# Shotgun analysis with HUMAnN2

## What is shotgun data?

##### shotgun sequencing = metagenomic sequencing = sequencing all of the DNA present in a sample

And we do mean ALL of the data - not just the gut bacteria, but the rest of the gut microbes (viruses, protists, Archaea, fungi, etc), things the host recently ate, and the host itself.

In the microbiome world, we use shotgun data to get a picture not only of what microbes are there, but what those microbes are doing. 16S rRNA sequencing is great at telling us who is there (or at least how many of each taxa are there). But it's not so good at telling us what functional roles those bacteria have. But if we have all (or many) of the genes present in the gut microbial community, we can get a clearer picture of what functions are present. So, we can ask "what is my microbial community doing (or able to do)?" 

Note that this does not necessarily give us an unambiguous idea of which bacteria have each function. Deep shotgun sequencing can help us address this problem by sequencing to a depth where we are able to pull out most of the genes in a sample. But without an enormous sequencing depth and an ability to extract and build a DNA library that actually contains all of the genes in a metagenome, we are never going to be able to capture complete genomes for all of the bacteria in a microbiome.

For basic analysis and function profiling of shotgun data, we will use HUMAnN2. HUMAnN2 is part of the bioBakery suite of bioinformatics tools for microbial community profiling. bioBakery includes tools both for the initial sequence (or other -omics) data, and for downstream analysis. You can read more about HUMAnN2 [here](https://bitbucket.org/biobakery/humann2/wiki/Home). More about the bioBakery project can be found [here](https://bitbucket.org/biobakery/biobakery/wiki/Home). bioBakery is developed by Curtis Huttenhower's [lab](http://huttenhower.sph.harvard.edu/) at Harvard.


## First! Data cleaning

You might have noticed above that HUMAnN2 takes quality-filtered sequence data as input. We do not get quality-filtered data back from the sequencing center. The raw sequence reads we get from the sequencing center still include the sequencing adapters, low quality reads, portions of reads that are low-quality, and host sequences (which are sometimes useful, but not when you are trying to determine microbial function). There are a lot of sequence trimming/cleaning/filtering tools out there, but we are going to use the tool that exists within the bioBakery suite - [KneadData](https://bitbucket.org/biobakery/kneaddata/wiki/Home)

#### Kneaddata:
1. Removes reads that match a user-specified database. For us, we are interested in removing host reads, so we use a human genome database. Most primate reads will match this database well enough to be filtered out. If you have another known source of contamination in your data, you can create a custom database that KneadData will use. KneadData uses [Bowtie2](http://bowtie-bio.sourceforge.net/bowtie2/manual.shtml) for this step by default, but can also use BMTagger if you have a strong preference for an alternate short-read aligner. You can change the default options for Bowtie2 if you are so inclined.
2. Removes low-quality reads, truncates reads once the quality dips below a certain threshold, removes short reads, and removes sequencing adapters. KneadData uses [Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic) for this. You again have lots of control over the options.

### We will first run KneadData on one file so we can look at the output:

Open up your command line utility, and log into Quest

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

Run KneadData on one (small) file and output the results to a folder within your personal folder in the class allocation

-----
```
cd /projects/e30740
module load java
kneaddata --input /projects/e30740/humann2_tutorial_seqs/SRR1761726_1.fastq.gz --reference-db /projects/e30740/humann2_ref_data/Homo_sapiens --output /projects/e30740/<yourfolder>/kneaddata_output
```
-----

Once is completes, exit out of the interactive job

-----
```
exit
```
-----

Open CyberDuck, navigate to /projects/e30740/<yourfolder>/kneaddata_output, and download the results files. There should be 4 files:
    SRR1761726_1_kneaddata_Homo_sapiens_bowtie2_contam.fastq
    SRR1761726_1_kneaddata.fastq
    SRR1761726_1_kneaddata.trimmed.fastq
    SRR1761726_1_kneaddata.log
    
The first file (the contam.fastq file) are reads that hit your reference database (e.g., the human reads).
The second file is your trimmed and quality-filtered output that contains no reads that hit the reference database. THIS IS THE OUTPUT FILE YOU WANT.
The third file are reads that are trimmed and quality-filtered, but still contain the contaminating reads.
The fourth file is a log file.

Go ahead an open up the log file on your computer. It will have a lot of details about what KneadData actually did, which can be helpful. 

### We will then run KneadData as a batch job

So, this would be tedious to do repeatedly for more than like 5 files, right? That is why computer science gave us for loops. We are going to create a bash script that submits the same job script over and over, but swaps out the input file everytime. That way, we can run all of our files through KneadData simultaneously.

Back in your command line, navigate to your home directory.

-----
```
cd /home/<yournetid>
```
-----

Copy (not move, copy) the example KneadData submission script from the class allocation to your home directory.

-----
```
cp /projects/e30740/kneaddata.sh /home/<yournetid>
```
-----

Use nano to open it

-----
```
nano kneaddata.sh
```
-----

See where it says "[P1]"? That is a variable. What gets entered in place of that variable is specified in a parameter file (named params.txt in this script). All that parameter file contains is a list of the files you are interested in running this script on.

While we are looking at the script, lets change <yourfolder> to your personal folder's actual name and the email address to your email.

Ok, close kneaddata.sh by pressing ctrl-X and then y then enter.

Let's create that parameter file in your home directory.

-----
```
ls /projects/e30740/humann2_tutorial_seqs > /home/<yournetid>/params.txt
```
-----

Open it with nano and make sure it makes sense.

-----
```
nano params.txt
```
-----

Sometimes (and I'm sure there's a logic behind this, but I haven't investigated what it is yet), ls results in the full filepath being listed and sometimes it results in only the filename being appended. This doesn't matter a ton, but will matter for how you structure the input filename in your script. Repeating the filepath twice won't make sense to KneadData.

Anyway, on to submitting your jobs! When submitting jobs via a script that loops through the job submission script, we don't use msub. We just execute the script.

So, make the script executable

-----
```
chmod u+x /home/<yournetid>/kneaddata.sh
```
-----

Run the script

-----
```
kneaddata.sh
```
-----

You should now see all of your jobs submitting one by one. And, soon, you will get an email for each job you have submitted! Keeping these emails is encouraged, in case you need to track errors later.

##### Note! Quest is currently switching to a new scheduler (the software that determines job priority and actually tells the jobs to run). Once we switch to the new scheduler, it may become preferable to use something called a job array to submit multiple jobs at once instead of using for loops. 

## Now! What is HUMAnN2 actually doing?

HUMAnN2 creates a functional profile of a microbial community. It determines both the presence/absence and abundance of microbial pathways. When it can, it also assignes a pathway to a bacterial taxon. 

#### What HUMAnN2 is doing:

1. Starts with quality-filtered sequence reads
2. Does some basic taxonomic profiling (that speeds up some steps further on)
3. Maps all of your reads to a large reference database of species pangenomes (species pangenome = all of the genes for all of the known strains of a particular species of bacteria)
4. For any reads that do not match anything in the species pangenome database, it then searches against a protein reference database
5. Combines the results from the search against the pangenome database, the protein reference database, and known bacterial functional pathways (using some fancy algorithms)
6. Spits out three results tables: gene family abundance, pathway abundance, and pathway coverage. 

The HUMAnN2 documentation has a helpful diagram:

![humann2_diamond_500x500.jpg](attachment:humann2_diamond_500x500.jpg)

If you happen to have alignment results or a gene table already lying around, you can also start mid-workflow.

Note: HUMAnN2 (and KneadData) can be installed on any Linux or Mac computer, but requires a lot of dependencies, so we will do everything on Quest. On Quest, HUMAnN2 can be run directly with a lot of modifications to your path variables, or it can be run through a singularity container (like we did with QIIME2). There are drawbacks to the singularity container, but for the purposes of this class, the singularity container will do.

## Running HUMAnN2

The general steps we will take when running HUMAnN2 is to run the basic script on all of our quality-filtered files, then combine the resulting output tables, manipulate the tables a little (to stratify/unstratify and/or normalize the results - we will cover this next week), and then run various statistical analyses of interest on the tables.

When running HUMAnN2, you have a choice of which translated protein database to use. There is detailed information [here](https://bitbucket.org/biobakery/humann2/wiki/Home#markdown-header-selecting-a-level-of-gene-family-resolution) on which protein reference database might be best for your samples. When analyzing nonhuman primate samples, their gut bacteria are not necessarily well characterized, we use UniRef50. You can probably get away with UniRef90 for most human gut metagenomes, at least in WEIRD populations.

We are going to do something similar to what we just did with KneadData, where we submit a number of files as a batch job. HUMAnN2 takes a little longer to run than KneadData, so things are not going to complete before the end of class...

Navigate to your home directory.

-----
```
cd /home/<yournetid>
```
-----

Copy (not move, copy) the example HUMAnN2 submission script from the class allocation to your home directory.

-----
```
cp /projects/e30740/humann2.sh /home/<yournetid>
```
-----

Use nano to open it

-----
```
nano humann2.sh
```
-----

Note, there is an extra line in this script... the sleep command here is causing the script to wait a random number of seconds before starting. That is because we can't all access the same singularity container at the exact same moment.

Change <yourfolder> to your personal folder's actual name and the email address to your email. Close nano.

Create the parameter file in your home directory.

-----
```
cd /projects/e30740/<yourfolder>/kneaddata_output
ls *_kneaddata.fastq > /home/<yournetid>/params.txt
cd
```
-----

Open it with nano and make sure it makes sense. 

-----
```
nano params.txt
```
-----

Make the submission script executable

-----
```
chmod u+x /home/<yournetid>/humann2.sh
```
-----

Run the script

-----
```
humann2.sh
```
-----

All of your jobs will now submit! You'll get submission emails soon, and then completion emails over the next few days... some of these files will take up to 2 days to finish.

### HUMAnN2 output files

HUMAnN2 has three main output files (and a lot of intermediate files). 

Gene families files - this will be the abundance of groups of evolutionarily-related protein coding sequences with similar functions.
Pathway abundance files - this will be the abundance of pathways for particular reactions, adjusted for the number of genes in a pathway.
Pathway coverage files - this will be an estimate of pathway presence/absence, and does not take into account how abundance various genes in the pathway are.

Among the intermediate files you will find various alignments produced by HUMAnN2, Metaphlan results, and a log file. 

