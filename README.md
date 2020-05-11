# kraken2_SOPv1
An example protocol in markdown; adapted from K2-SOP.docx

## Disclaimer

The material embodied in this repository is provided to you "as-is" and without warranty of any kind, express, implied or otherwise, including without limitation, any warranty of fitness for a particular purpose. In no event shall the National Institutes of Health (NIH) or the United States (U.S.) government be liable to you or anyone else for any direct, special, incidental, indirect or consequential damages of any kind, or any damages whatsoever, including without limitation, loss of profit, loss of use, savings or revenue, or the claims of third parties, whether or not NIH or the U.S. government has been advised of the possibility of such loss, however caused and on any theory of liability, arising out of or in connection with the possession, use or performance of this software.

Disclaimer modified from: https://github.com/CDCgov/template/blob/master/DISCLAIMER.md

## Dependencies and Versions

kraken 2.0.8-beta

## Introduction

Kraken2 is one of many tools available for classifying metagenomic reads. It is part of a family of software including:

* Kraken1 - https://github.com/DerrickWood/kraken
* Kraken2 - https://github.com/DerrickWood/kraken2
* Krakenuniq - https://github.com/fbreitwieser/krakenuniq
* Bracken - https://github.com/jenniferlu717/Bracken

Kraken2 is an improvement of Kraken1 and uses a different database structure (their databases are not compatible). Krakenuniq is a fork of Kraken1 that uses unique kmers to reduce false positives (e.g., thinking you have B. anthracis when it’s really B. cereus). Bracken is a Kraken1/2 compatible Bayesian re-estimation framework that, conceptually, tries to do something similar to what the Pathoscope algorithm does (e.g., promotes reads to more specific nodes on the taxonomic tree). The Kraken1 and Kraken2 algorithms are currently maintained by different labs (Salzburg and Langmead), respectively.

Kraken2 was chosen over other algorithms because it is fast, easy to use and has a decent database builder. It suffers from not being vigorously under development (per Ben Langmead) but there are plans to maintain it and add features (like the Krakenuniq functionality). It also requires more memory (64G+) than conventional laptops but is easily run on the Biowulf HPC cluster, where it can be loaded using: module load kraken/2.0.8-beta .

The kraken algorithm is best explained by in the published manuscripts, accessible from the github repositories. Briefly, it assigns kmers from whole genomes in the reference database to a taxonomic hierarchy. More unique kmers are towards the tips, while common kmers are internal to the tree. Reads to be analyzed are broken into kmers and the location of each on the taxonomic hierarchy is collected. For instance, some typical read assignment strings are described below. The following kraken2 command was run on some simulated data (wgsim) generated from a set of reference genomes:

````
$ kraken2 -db /data/Segrelab/data/kraken_db/20200211_standard+_kraken2 --confidence 0.1 --paired --threads 4 --output new.txt --report new.report.txt sim_reads.r1.fq sim_reads.r2.fq
````
## Unclassified Reads, False Positives and Confidence

A. thaliana is not in the specified kraken2 database so reads generated from A. thaliana should be (U)nclassified and that’s is true for the vast majority of the data (first example). The read that is classified (second example) gets called taxid=131567=”Cellular Organisms” and it is a good example of how the algorithm works:

````
U       Atha_00001_6300076_6300363_1:0:0_0:0:0_10ff     0       150|150 0:116 |:| 0:116

C       Atha_00001_15377904_15378137_2:0:0_2:0:0_1100   131567  150|150 0:113 5820:2 0:1 |:| 0:7 131567:5 0:7 131567:14 0:28 208452:2 131567:5
 59732:1 131567:9 5820:5 0:2 5820:2 0:5 208452:2 0:22
````

Breaking that down, read pair Atha_00001_15377904_15378137_2:0:0_2:0:0_1100  is (C)lassified as taxid 131567 and the length of the reads in the pair is 150|150. The rest of the string is a space-delimited list of kmer composition:

* 0:113		The first 113 kmers of Read 1 are not in the database
* 5820:2		The last 2 kmers of Read 1 are taxid=5820=plasmodium
* 0:1		The last kmer is not in the database (116 35-mers in a 150nt sequence)
* |:|		Read pair divider. The remaining kmers are for Read 2
* 0:7		Not in database
* 131567:5	Cellular organisms
* 0:7		Not in database
* 131567:14	Cellular organisms
* 0:28		Not in database
* 208452:2	Plasmodium coatneyi
* 131567:5	Cellular organisms
* 59732:1	Chryseobacterium
* 131567:9	Cellular organisms
* 5820:5		Plasmodium
* 0:2		Not in database
* 5820:2		Plasmodium
* 0:5		Not in database
* 208452:2	Plasmodium coatneyi
* 0:22		Not in database

Note that 185 kmers aren’t found in the database and 47 are at internal nodes which collapse to the unhelpful “Cellular organisms” taxon. The confidence set for this run was 0.1 which means it reports the most specific taxonomic level where at least 10% of kmers are at that taxonomic level or one of it’s children (“Cellular organisms”). The next best would be Plasmodium with 13 kmers, but that doesn’t pass the confidence criteria. If you remove the confidence criteria (not advisable), you suddenly have Plasmodium:

````
14:33 biowulf novaseq12$ sinteractive --mem=96g
…
14:34 cn3104 test$ module load kraken/2.0.8-beta
[+] Loading kraken  2.0.8-beta 
14:34 cn3104 test$ kraken2 -db /data/Segrelab/data/kraken_db/20200211_standard+_kraken2 --paired --threads 1 --output new_noconf.txt --report new_noconf.report.txt sim_reads.r1.fq sim_reads.r2.fq
Loading database information... done.
315760 sequences (94.73 Mbp) processed in 16.806s (1127.3 Kseq/m, 338.20 Mbp/m).
  270775 sequences classified (85.75%)
  44985 sequences unclassified (14.25%)
14:36 cn3104 test$ grep 'Atha_00001_15377904_15378137_2:0:0_2:0:0_1100' new_noconf.txt
C       Atha_00001_15377904_15378137_2:0:0_2:0:0_1100   208452  150|150 0:113 5820:2 0:1 |:| 0:7 131567:5 0:7 131567:14 0:28 208452:2 131567:5 59732:1 131567:9 5820:5 0:2 5820:2 0:5 208452:2 0:22
````

Since every kmer (without Ns) counts in the confidence calculations, even the ones that aren’t found in the database, you should make sure your reads are trimmed of adapter sequences.

## Classified Reads and False Positives

A second example is this string for a read pair derived from a genome (S. epidermidis ATCC12228) that is in the database. This is read the same way as above but taxid=1282=Staphylococcus epidermidis and taxid=1279=Staphylococcus.
````
C       Sepi_00001_1400665_1400959_1:0:0_3:0:0_8        1282    150|150 0:2 1279:5 1282:7 1279:5 1282:35 1279:7 1282:55 |:| 1282:51 0:32 1282:9 0:1 1282:1 0:7 1282:3 0:1 1282:11
````
The assignment below is one of the (minority) of reads that map uniquely to the reference genome (taxid=176280) and not an internal taxonomic node.
````
C       Sepi_00001_1199230_1199479_1:0:0_0:0:0_3b1      176280  150|150 1282:116 |:| 1282:11 1279:3 1282:5 1279:6 1282:2 1385:5 176280:34 1282:50
````
For 10,000 simulated reads (with 1% sequencing error) generated from the S. epidermidis ATCC12228 genome, this is the breakdown of read classifications:

````
Count	Percent	Taxid	Taxon
9489	94.89%	1282	Staphylococcus epidermidis
413	4.13%	1279	Staphylococcus
36	0.36%	2	Bacteria
12	0.12%	176280	Staphylococcus epidermidis ATCC12228
11	0.11%	91061	Bacilli
8	0.08%	1385	Bacilliales
5	0.05%	1280	Staphylococcus aureus
5	0.05%	1	Root
4	0.04%	90964	Staphylococcaceae
4	0.04%	0	Not found
2	0.02%	46126	Staphylococcus chromogenes
2	0.02%	1239	Firmicutes
1	0.01%	985762	Staphylococcus agnetis
1	0.01%	46170	Staphylococcus aureus subsp aureus
1	0.01%	218284	Bacillus vietnamensis
1	0.01%	186817	Bacillaceae
1	0.01%	1783272	Terrabacteria group
1	0.01%	1449752	Staphylococcus epidermidis PM221
1	0.01%	1294	Staphylococcus muscae
1	0.01%	1292	Staphylococcus warneri
1	0.01%	1290	Staphylococcus homins
````
So, even for an ideal simulated read set 13/10000 reads are assigned to the wrong species and only about 95% are classified to the species level. These toy examples explain how kraken2’s taxonomic assignment works and the pitfalls of trusting calls with very low numbers of reads.

## Extended usage info

Kraken2 usage is pretty straight-forward and the online manual is an excellent place to start:
https://github.com/DerrickWood/kraken2/blob/master/docs/MANUAL.markdown

This is a typical kraken2 commandline:
````
kraken2 --db /data/Segrelab/data/kraken_db/20200211_standard+_kraken2 --confidence 0.1 --paired --threads 4 --output output.txt --report k2report.txt myreads.R1.fq myreads.R2.fq
````
This typically needs to be run on a machine with ~64g of memory and multiple threads. Something like: sinteractive --mem=96g --cpus-per-task 8 or via the swarm command like: swarm -f myswarm -g 96 -t 8

Some additional flags you might be interested in:

* --unclassified-out FILENAME will return “Dark Matter” that you can analyze further. Make sure to include a “#” in FILENAME for paired reads since this is where kraken2 will stick the read signifier (“_1” or “_2”)
* --confidence 0.1 Sets the confidence to 0.1. This is explained in the kraken2 manual and an example of what happens if you don’t set confidence is described in the Introduction above. I’m not convinced that this is the best metric for excluding spurious identifications (and neither are the authors), but it is better than nothing. One issue is that kmers not found in the database are included in the denominator of the confidence threshold calculation.

## Input data
Kraken2 takes read files, paired or single. For paired reads you will need to extract them in such a way that you have exactly the same reads in the same order for the two mates. Most of our metagenomic samples are stored as interleaved reads with or without singletons. You can get some information about a fasta using the inspect_fastx.pl script. The most useful column is 'pairing' which will tell you if paired reads are R1/R2, interleaved (1,2,1,2,1,2...) or mixed (1,2,1,1,2,1,1...). It only looks at the first and last parts of the file to speed things up. Unfortunately, compressed read files slow things down because it needs to look through the entire uncompressed stream.
```` 
$ perl -w /data/Segrelab/bwbin/inspect_fastx.pl *.*
zip     type    pairing error   file
none    fastq   R1      false   BMC_MET0200.nsrt.nohum.r1.fq
none    fastq   R2      false   BMC_MET0200.nsrt.nohum.r2.fq
none    fastq   R1      false   FMC_MET0201.bam.pbhuman.r1.fq
none    fastq   R2      false   FMC_MET0201.bam.pbhuman.r2.fq
none    undef   undef   false   MET1687.bam.logfile
gzip    fastq   mixed   false   MET1687.nohuman.fq.gz
none    undef   undef   false   MET1688.logfile
gzip    fasta   inter   false   MET1688.no_human.p.fa.gz
gzip    fasta   mixed   false   MET1688.no_human.s.fa.gz
gzip    fastq   mixed   false   Met2615.nohuman.fq.gz
gzip    fastq   mixed   false   Met2817.qc.nohuman.fastq.gz
none    undef   undef   false   MockTestData.tar
````
For both mixed and interleaved files you can usually extract the paired R1 and R2 files using the following script:
````
$ /data/Segrelab/bwbin/novaseq_fqsplit.pl -in MET1172.no_human.paired.fasta.screen.fastq -read 1p > read1.fq
$ /data/Segrelab/bwbin/novaseq_fqsplit.pl -in MET1172.no_human.paired.fasta.screen.fastq -read 2p > read2.fq
````
You should sanity check that the split read files have the same number of lines. Also, you can leave the source file compressed and just pipe it through novaseq_fqsplit.pl to same space.

## Output data

The primary output from kraken2 is a per-read classification (--out) and a taxonomy report (--report). Examples:
```
--out
U       Atha_00001_19477053_19477357_3:0:0_1:0:0_0      0       150|150 0:116 |:| 0:116
U       Atha_00001_13742298_13742594_3:0:0_1:0:0_1      0       150|150 0:116 |:| 0:116
U       Atha_00001_9029161_9029357_0:0:0_3:0:0_2        0       150|150 0:116 |:| 0:116

--report
 20.51  64771   64771   U       0       unclassified
 79.49  250989  1542    R       1       root
 63.45  200349  55      R1      131567    cellular organisms
 57.18  180557  559     D       2           Bacteria
 44.41  140225  38      D1      1783272       Terrabacteria group
 27.97  88317   20      P       1239            Firmicutes
 27.96  88297   138     C       91061             Bacilli
 24.77  78205   43      O       1385                Bacillales
 21.60  68210   35      F       90964                 Staphylococcaceae
 21.59  68175   2435    G       1279                    Staphylococcus
```
The per-read output is described in the introduction. The report file is self-explanatory, but may cause some confusion for users of Clinical Pathoscope. Kraken2 can place reads at any level of the taxonomic hierarchy. In the example above, 88,297 reads are assigned to Bacilli or one of it’s child taxonomic nodes. Only 138 are assigned directly to Bacilli, the remaining reads are assigned to a more specific taxonomic label. Clinical Pathoscope is designed to distribute these “last common ancestor” reads to the species level. Bracken can be used to implement a similar functionality.

## Shortcuts (not recommended)

* Kraken2 doesn’t have the “Minikraken” database that was suggested for running kraken1 in low resource environments. It does have instructions for replicating that behavior by adjusting options in the kraken2-build process. You don’t need to do this on Biowulf. Just ask for more memory.
* I’ve never tried the --quick option of kraken2 because I’ve never needed to on Biowulf. Just request more threads.
