# Introduction

MicroExonator is a fully-integrated computational pipeline that allows for systematic de novo discovery and quantification of microexons using raw RNA-seq data for any organism with a gene annotation. Compared to other available methods MicroExonator is more sensitive for discovering smaller microexons and it provides higher specificity for all lengths. Moreover, MicroExonator provides integrated downstream comparative analysis between cell types or tissues using [Whippet](https://github.com/timbitz/Whippet.jl) ([Sterne-Weiler et al. 2018](https://doi.org/10.1016/j.molcel.2018.08.018)). As a proof of principle MicroExonator  identified X novel microexons in Y RNA-seq samples from mouse early development to provide a systematic characterization based on time and tissue specificity.


# Installation

Start by cloning MicroExonator

    git clone https://github.com/hemberg-lab/MicroExonator

Install [Miniconda 2](https://docs.conda.io/en/latest/miniconda.html)

    wget https://repo.continuum.io/miniconda/Miniconda2-latest-Linux-x86_64.sh
    chmod +x Miniconda2-latest-Linux-x86_64.sh
    ./Miniconda2-latest-Linux-x86_64.sh -b -p /cvmfs/softdrive.nl/$USER/Miniconda2

After the release of conda 4.4 some additional configuration might be required to work with snakemake worflow that have fix conda enviroments, particulaly there might be some inteference of some conda enviroments when that pipeline call is done from an screen [interference with screen comand](https://stackoverflow.com/questions/50591901/screen-inside-the-conda-environment-doesnt-work). Thus, we recomend modify your `~/.bashrc` file and add the following line:

    . /path/to/miniconda/etc/profile.d/conda.sh

Where `/path/to/miniconda` is the directory where you installed Miniconda, which is at your home directory by default, but during Miniconda instalation can be set at any directory.

Finally, create an enviroment to run [snakemake](https://snakemake.readthedocs.io/en/stable/)

    conda create -n snakemake snakemake python=3.6 pandas cookiecutter

# Configuration

Before running MicroExonator you need to have certain files in the `MicroExonator/` directory. First, you need to create a `config.yaml`, which should contain the path of the input files and certain paramethers. The format for the `config.yaml` file is shown below:

    Genome_fasta : /path/to/Genome.fa
    Gene_anontation_bed12 : /path/to/ensembl.bed12
    GT_AG_U2_5 : /path/to/GT_AG_U2_5.good.matrix
    GT_AG_U2_3 : /path/to/GT_AG_U2_3.good.matrix
    conservation_bigwig : /path/to/conservation.bw  
    working_directory : /path/to/MicroExonator/
    ME_DB : /path/to/ME_DB.bed
    ME_len : 30
    Optimize_hard_drive : T

Here:

* `Genome_fasta` is a [multifasta](http://www.metagenomics.wiki/tools/fastq/multi-fasta-format) file containg the cromosomes. 
* `Gene_anontation_bed12` is a BED file containing the transcript annotation. A collection of these files can be found at [UCSC genome browser](http://genome.ucsc.edu/cgi-bin/hgTables). 
* `GT_AG_U2_5` and `GT_AG_U2_5` are splice site PWMs that can come from [SpliceRack](http://katahdin.cshl.edu/SpliceRack/poster_data.html) (as this server is currently down, we provisionally provide PWMs for human and mouse), but if you do not have these PWMs for the species you are interested in, you can leave it as `NA` and MicroExonator will generate the PWM internally based on annotated splice sites.
* `conservation_bigwig` is a bigwig file containing genome-wide conservation scores generated by Pylop or PhastCons, which can be downloaded from [UCSC genome browser](http://hgdownload.cse.ucsc.edu/downloads.html) for some species. If you do not have a `conservation_bigwig` file, you need to create an bigwig which has the value 0 for every position in your genome assembly.
* `working_directory` is the path to the MicroExonator folder that you are working with
* `ME_len` is microexon maximum lenght. The default value is 30.
* `Optimize_hard_drive` can be set as `T` or `F` (true of false). When it is set as `F`, fastq files will be downloaded or copied only once at `FASTQ/` directory. This can be inconvenient when you are analysing large amout of data (fastqs >1TB), because the copy will not be deleted until MicroExonator finish completely. When `Optimize_hard_drive` is set as `T` instead, two independet local copies will be generated for the discovery and quantification moduled. As these are temporary copies, every copied fastq file will be deleted as soon as is mapped to the splice junction tags, which means the disk space usage will be set to the minimum while MicroExonator is runing. 

To imput the RNA-seq data, you need to either create a `design.tsv` (for fastq.gz files that are stored locally) and/or `NCBI_accession_list.txt`(for SRA accession names) which are automatically downloaded. You can find examples of these files inside the Examples folder. 

Finnaly, if you are working on a high performace cluster, then it is very likely that you need to submit jobs to queueing systems such as lsf, qsub, SLURM, etc. To make MicroExonator work with these queueing systems, you need to create a `cluster.json` file. We currently provide in the Examples folder a `cluster.json` file to run MicroExonator with [lsf](https://www.ibm.com/support/knowledgecenter/en/SSETD4/product_welcome_platform_lsf.html). To adapt MicroExonator to other quequing systems please see the [SnakeMake documentation](https://snakemake.readthedocs.io/en/stable/snakefiles/configuration.html?highlight=cluster.json#cluster-configuration).



# Running

We highly recommend creating a screen before running MicroExonator

    screen -S session_name  #choose a meaning full name to you

To activate snakemake enviroment

    source activate snakemake

Then run

    snakemake -s MicroExonator.skm  --cluster-config cluster.json --cluster {cluster system params} --use-conda -k  -j {number of parallel jobs}

Notice that you should use `--cluster` only if you are working in a computer cluster that uses a queuing systems. We provide an example of `cluster.json` to work with lsf and in that case the cluster system params should be `"bsub -n {cluster.nCPUs} -R {cluster.resources} -c {cluster.tCPU} -G {cluster.Group} -q {cluster.queue} -o {cluster.output} -e {cluster.error} -M {cluster.memory}"`. The number of parallel jobs can be a positive integer, the appropriate value depends on the capacity of your machine but for most users a value between 5 and 50 is appropriate. 

If you want to process a large dataset, we recommend to set `Optimize_hard_drive` as `T`. If you tho this, you can also run the pipeline in two steps:

    snakemake -s MicroExonator.skm  --cluster-config cluster.json --cluster {cluster system params} --use-conda -k  -j {number of parallel jobs} discovery

Once the pipeline finish the discovery module, it can be resumed any time later as:

    snakemake -s MicroExonator.skm  --cluster-config cluster.json --cluster {cluster system params} --use-conda -k  -j {number of parallel jobs} quant

If you are working remotelly, the connection is likely to die before MicroExonator finish. However, as long as you are working within an screen, loggin off will not kill snakemake. To list your active screens you can do:

    screen -ls
 
To reattached and detach screens just use:

    screen -r session_name  # only detached screen can be reattached  
    screen -d session_name

# Troubleshooting

Before running it is recommended to check if SnakeMake can corretly generate all the steps given your input. To do this you can carry out a dry-run using the `-np` parameter:

    snakemake -s MicroExonator.skm  --cluster-config cluster.json --cluster {cluster system params} --use-conda -k  -j {number of parallel jobs} -np

The dry-run will display all the steps and commands that will be excecuted. If the dry-run cannot be initiated, make sure that you are running MicroExonator from inside the folder you cloned from this repository. Also make sure you have the right configuration inside `config.yaml`. 

If you experice any errors during the pipeline run, we recomend to check the error logs. As long as the snakemake call was not interrupted, you can allways resume the run, as snakemake keep track of any error and is able keep track of the complete and incomplete steps. To make sure snakemake will not overide already completed files, do a dry-run call before resuming MicroExonator. If on the dry-run you realise some steps are going to be re-runed, remove `FASTQ\` directory from MicroExonator folder.

# Contact

For questions, ideas, feature requests and potential bug reports please contact gp7@sanger.ac.uk.
