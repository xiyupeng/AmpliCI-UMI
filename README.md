AmpliCI-UMI
===========

AmpliCI-UMI, Amplicon Clustering Inference with UMI information, is built on [AmpliCI](https://github.com/DormanLab/AmpliCI), but takes advantage of UMI information for denoising amplicon sequencing data, which contributes to a more accurate error profile and better clustering results.
AmpliCI-UMI greatly enhances the accuracy of detecting rare sequences and provides deduplicated abundance estimation, elimilating PCR-induced errors and bias.


# Table of Contents
1. [Installation](#installation)
1. [Preparing input](#input)
1. [Usage](#usage)
1. [Tunning Parameter](#parameter)
1. [Output](#output)
1. [Options](#options)
1. [Acknowledgments](#acknowledgments)
1. [Citation](#citation)
1. [Contact](#contact)


# Installation <a name = "installation" />

AmpliCI-UMI shares the same prerequisites with AmpliCI. Please check the [prerequisites](https://github.com/DormanLab/AmpliCI#prerequisites) before the following installation. 
The software has been tested under Linux and MacOS.

1. Clone the repository.

    ```sh
    git clone https://github.com/xiyupeng/AmpliCI-UMI.git
    ```

2. Configure the project.

   ```sh
   cd AmpliCI-UMI/src
   cmake .
   ```

3. Compile AmpliCI.  The executable is called ```run_AmpliCI```.  It will appear in the ```src``` directory you are currently in.

   ```sh
   make
   ```

# Preparing input <a name="input" />

The input of AmpliCI is a FASTQ file. 
Besides the common practice for [preprocessing](https://github.com/DormanLab/AmpliCI#input) raw FASTQ files,
additional preprocessing steps are required for AmpliCI-UMI.
For each sample, users need to prepare their data in three FASTQ files.

- `FILENAME.bc.fq`

- `FILENAME.trim.fq`

- `FILENAME.fq`


Each FASTQ file contains subsequences of the original FASTQ file, UMIs (`FILENAME.bc.fq`), biological sequences (`FILENAME.trim.fq`) or UMIs + biological sequences (`FILENAME.fq`).
Subsequences from the same read share the same seq ID and are in the same order in all three files.
In `FILENAME.fq`, UMIs should be at the front of biological sequences.
It is easy to prepare the three FASTQ files using [seqkit](https://github.com/shenwei356/seqkit) with subcommand `subseq` and `concat`.

Below is an example how to prepare the other two files from `FILENAME.fq` (total reads length 250nt with UMI length 9nt)

```
seqkit subseq -r 1:9 FILENAME.fq > FILENAME.bc.fq
seqkit subseq -r 10:250 FILENAME.fq > FILENAME.trim.fq
```

or generate `FILENAME.fq` based on the other two files

```
seqkit concat FILENAME.bc.fq FILENAME.trim.fq > FILENAME.fq 
```


# Usage <a name="usage" />

AmpliC-UMI contains four major step in the pipeline.

1. Cluster UMI sequences (the executable is called run_AmpliCI):

```sh
./run_AmpliCI --umi --fastq <input_umi_fastq_file> --outfile <output_umi_base_filename>
```

An example (from the ```src``` directory):

```sh
./run_AmpliCI --umi --fastq ../test/sim2.bc.fq --outfile ../test/sim2.bc
```

2. Estimate error profile based on partitions indicated by umi clusters:

```sh
grep "assign" <input_umi_out_file> | awk '{$1="";print}' > <output_umi_partition_file>
./run_AmpliCI --fastq <input_trim_fastq_file>  --outfile <output_error_profile_file> --partition <input_umi_partition_file> -exclude --abundance 2 --error 
```

An example:

```sh
grep "assign" ../test/sim2.bc.out | awk '{$1="";print}' > ../test/partition.txt
./run_AmpliCI --fastq ../test/sim2.trim.fq  --outfile ../test/error.out --partition ../test/partition.txt -exclude --abundance 2 --error
```

3.  Cluster umi-tagged sequences to get initial haplotype set ([seqkit](https://github.com/shenwei356/seqkit) required for truncation and deduplication):

```sh
./run_AmpliCI --fastq <input_merge_fastq_file> --outfile <output_merge_base_filename> --profile <input_error_profile_file> -trim <umi_length>
cat <input_merge_fasta_file> |  seqkit subseq -r <start_idx>:<end_idx> | seqkit rmdup -s | seqkit seq -w 0 > <output_hap_fasta_file>
```

`<start_idx>` and `<end_idx>` are start and end position of biological sequences.
Thus UMI length is `<start_idx>-1` and biological sequence length is `<end_idx>-<start_idx>+1`.

An example (total reads length 250nt with UMI length 9nt):

```sh
./run_AmpliCI --fastq ../test/sim2.fq --outfile ../test/sim2.merge --profile ../test/error.out -trim 9 
cat ../test/sim2.merge.fa |  seqkit subseq -r 10:250 | seqkit rmdup -s | seqkit seq -w 0 > ../test/sim2.merge.trim_dedup.fa
```

4. Estimate deduplicated abundance of each haplotype. rho is a tunning parameter. We describe how to tun this parameter in the following section.

```sh
./run_AmpliCI --fastq <input_merge_fastq_file> --umifile <input_umi_fasta_file> --haplotype <input_hap_fasta_file> -umilen <umi_length> --outfile <output_base_filename> --profile error.out -rho <rho>
```

An example:

```sh
./run_AmpliCI --fastq ../test/sim2.fq --umifile ../test/sim2.bc.fa -haplotype ../test/sim2.merge.trim_dedup.fa -umilen 9 --outfile ../test/test --profile ../test/error.out -rho 46
```

- You can run the whole pipeline on this example (from the ```src``` directory):

```sh
bash ../script/ana.bash
```

- Alternative step to the 3rd step. The main goal of the 3rd step is to initialize a haplotype set. 
One alternative step could be directly running AmpliCI on non-UMI-tagged fastq files.

```sh
./run_AmpliCI --fastq <input_trim_fastq_file> --outfile <output_hap_base_filename> --profile <input_error_profile_file>
```
The alternative step costs less time but with a risk of missing some very similar haplotypes.
We recommend to run the alternative step on massive datasets with sequences with moderate similarity, e.g. 16S rRNA gene sequences.

- More detailed help can be obtained with:

```sh
./run_AmpliCI --help
```

# Tunning Parameter <a name = "parameter" />

A separate program will be used to find the value of the tuning parameter rho. 
You can find the program from the ```script``` directory.

1. **Preparation**

First, codes can be compiled with
```
cd ../script
gcc -Wall -pedantic -Wextra -g -o bp_pmf_mix bp_pmf_mixture.c fft.c -lfftw3 -lm
```
You may need to add additional path to the Rmath library (use -L) and header files (use -I) if Rmarh is not on your default path.
The input file is the count distribution of UMI raw abundance in the sample, 
that the number in the ith row is the number of unique UMIs with abundance i.
You can find examples of the input file `.txt` under the ```script``` directory.

2. **Model Fitting**

The first step is to fit our proposed model (details in the paper) to data (the executable is called bp_pmf_mix).
```
./bp_pmf_mix -f  <input_data_file>  -t <truncate_position> > <output_csv_file>
```
You can use option `-t` to select a truncate position to truncate the long right tail of the distribution, which may be contributed by unmodeled UMI collision.

3. **Refit Model**

You can also use the estimated values of parameters as the input to the program, in order to obtain the desired distribution without reestimating parameters.
Below we use the input file from a HIV dataset as an example. 
```
./bp_pmf_mix -f hiv1_raw_abun.txt -t 100 --efficiency 0.6 --ncycles 11 --epsilon 0.004 --delta 0.01 > text.csv
```
Options `--efficiency`, `--ncycles`, `--epsilon` and `--delta` are four parameters in the model, that represent PCR efficiency, number of PCR cycles, PCR error rate and sequence error rate. You can find more details about the model intepretation in the paper. 

4. **Select Rho**

In practice, we currently choose rho as the argmax {Pr(X <= rho | Z = 1) < 0.05},
which means at most five percent of true variants will have observed abundance below or equal to rho.
And you can find such information at the 8th column in the output csv file.

# Output Files <a name = "output" />

The final step of the pipeline with argument `--outfile <output_base_filename>` generates two output files:

***1.`output_base_filename.fa`***

FASTA-formatted file containing denoised sequences (or haplotypes) as well as their deduplicated abundances. For example, for the first haplotype, the FASTA header might look like:

```
>H0;Deduplicated Abundance=18.000;
```

For each haplotype, deduplicated abundance is the estimated number of unamplified sample sequences.


***2.`output_base_filename.out`***

A text file with the following information provided as key: value pairs, one per line.  The keys are:

- `log likelihood`: Penalized log likelihood of the model. See the paper for more details.

- `K`: Number of haplotypes (total number of sequences in the input haplotype set <input_hap_fasta_file> )

- `assignments`: model-assigned haplotype for each read in FASTQ-determined input order.  Haplotypes are numbered 0, 1, ..., and match the sequences H0, H1, ... in the output FASTA file of haplotypes.

- `cluster sizes`: Number of reads assigned to each haplotype.

- `UMI K`: Number of unique UMIs (total number of sequences in the input UMI set <input_umi_fasta_file> )

- `UMI assignment`: model-assigned UMI for each read in FASTQ-determined input order.

- `UMI cluster sizes`: Number of reads assigned to each UMI.

- `reads ll`: The posterior log likelihood for each read.

- `Eta`: Relative abundance of each unique UMI

- `Gamma`: A UMI_K X K matrix used to indicate dependence between UMI and haplotypes. The deduplicated abundance of the kth haplotype is the kth column sum of the Gamma.

- There is also a list of haplotypes with related UMIs reported in this file.

The example below showed one haplotype with related 3 UMIs. Thus the deduplicated abundance of the haplotype is 3.  

```
AGGATTGATTAAATATTATTGTCCTATTGAAGTGTTCTCTCAATTTTTCACTTACTCTTTGTAAAGTTTTCTCCCATTTAGTTTTATTAATGTTACAATGTGCTTGTCTTATATCTCCTATTATGTCTCCTGTTGCATAGAATGTTTGTCCTGGTCCTATTCTTACACTTTTTCTTGTATTATTGTTGGGTCTTATACAATTAATCTCTACAGATTCATTGAGATGTACTATTATTGTTTT
TTTTAAAAC GTAAATAGT CGCTAATGA
```

# Options <a name = "options" />

Options of AmpliCI can be found in [here](https://github.com/DormanLab/AmpliCI#options). Below we list options specific for AmpliCI-UMI.

- `--partition`: Use a partition file when estimating errors. The partition file contains cluster assignment of each read. Reads assigned with the same number are in the same group. We can use either true partition or UMI-induced partition file to generate a better error profile. 

- `--exclude`: Exclude small clusters during error estimation (set threshold with option --abundance).

- `--abundance` under `error` estimation mode: Lower bound on observed abundance for inclusion of seeded cluster during error estimation.

- `--umi`: Used for clustering UMIs. Compared to the default, we set the gap score -20, and band width 2, used in Needleman Welch alignment. We also disable JC69 model since UMIs are random sequences.

- `--trim`:Ignore first # nucleotides in JC69 model. Since UMIs are random sequences, they should not be ignored when fit the JC69 model.

- `--umifile`: FASTA file with UMIs.

- `--rho`:  Tunning parameter that control the sparsity of the transition matrix `Gamma`. We have described how to select `rho` above.

- `--umilen`: Length of each UMI.
 


# Acknowledgments <a name = "acknowledgments" />

- See acknowledgments of [AmpliCI](https://github.com/DormanLab/AmpliCI#acknowledgements)


# Citation <a name = "citation" />

- Peng, X. and Dorman, K. (2020) ‘AmpliCI: A High-resolution Model-Based Approach for Denoising Illumina Amplicon Data’, Bioinformatics. doi: [10.1093/bioinformatics/btaa648](https://academic.oup.com/bioinformatics/article/doi/10.1093/bioinformatics/btaa648/5875058).

# Contact <a name = "contact" />

If you have any problems, please contact:

Xiyu Peng (pansypeng124@gmail.com)

Karin Dorman (kdorman@iastate.edu)
