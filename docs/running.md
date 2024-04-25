
# Running your analysis step by step

When the configuration files are ready, you can start the run by `sbatch Workflow.sh wgbs`.

```
[username@clust-slurm-client Methylator]$ sbatch Workflow.sh wgbs
```

Please see below detailed explanation. 

## FASTQ quality control (eventually after SRA data retrieval)

Prerequisite:   
- Using your own data: your FASTQ files are on the cluster, in our example in `/shared/projects/YourProjectName/Raw_fastq` (but you can name your folders as you want, as long as you adjust the `READSPATH` parameter in `config_main.yaml`). 
- You have modified `config/metadata.tsv` according to your experimental design (with sample names or SRR identifiers).

Now you have to check in `config/config_main.yaml` that: 

- you gave a project name

```yaml
# Project name
PROJECT: EXAMPLE
```

- In `Control of the workflow`, `QC` or `SRA` is set to `yes`:

If you want to download data from SRA, set `SRA` to `yes`. The QC will follow automatically. If you use your own data, put `QC` to `yes` and `SRA` to `no`.  

```yaml
## Do you want to download FASTQ files from public from Sequence Read Archive (SRA) ? 
SRA: no  # "yes" or "no". If set to "yes", the workflow will stop after the QC to let you decide whether you want to trim your raw data or not. In order to run the rest of the workflow, you have to set it to "no".

## Do you need to do quality control?
QC: yes  # "yes" or "no". If set to "yes", the workflow will stop after the QC to let you decide whether you want to trim your raw data or not. In order to run the rest of the workflow, you have to set it to "no".
```

The rest of the part `Control of the workflow` will be **ignored**. The software will stop after the QC to give you the opportunity to decide if trimming is necessary or not. 

- The shared parameters are correct (paths to the FASTQ files, metadata.tsv, result folders, single or paired-end data). 

```yaml
## the path to fastq files
READSPATH: /shared/projects/YourProjectName/Raw_fastq

## the meta file describing the experiment settings
METAFILE: /shared/projects/YourProjectName/configs/metadata.tsv

## paths for intermediate and final results
BIGDATAPATH: /shared/projects/YourProjectName/data # for big files
RESULTPATH: /shared/projects/YourProjectName/results
```

When this is done, you can start the QC by running:

```
[username@clust-slurm-client Methylator]$ sbatch Workflow.sh wgbs
```

You can check if your job is running using squeue.

```
[username@clust-slurm-client Methylator]$ squeue --me
```

You should also check SLURM output files. See [Description of the log files](#description-of-the-log-files). 

## FastQC results

If everything goes fine, fastQC results will be in `results/EXAMPLE/fastqc/`. For every sample you will have something like:

```
[username@clust-slurm-client Methylator]$ ll results/EXAMPLE/fastqc
total 38537
-rw-rw----+ 1 username username  640952 May 11 15:16 Sample1_forward_fastqc.html
-rw-rw----+ 1 username username  867795 May 11 15:06 Sample1_forward_fastqc.zip
-rw-rw----+ 1 username username  645532 May 11 15:16 Sample1_reverse_fastqc.html
-rw-rw----+ 1 username username  871080 May 11 15:16 Sample1_reverse_fastqc.zip
```

Those are individual fastQC reports. [MultiQC](https://multiqc.info/docs/) is called after FastQC, so you will also find `report_quality_control.html` that is a summary for all the samples. 
You can copy those reports to your computer by typing (in a new local terminal):

```
You@YourComputer:~$ scp -pr username@core.cluster.france-bioinformatique.fr:/shared/projects/YourEXAMPLE/Methylator/results/EXAMPLE/fastqc PathTo/WhereYouWantToSave/
```
or look at them directly in the Jupyter Hub.  
It's time to decide if how much trimming you need. Trimming is generally necessary with WGBS or RRBS data. 

<span>{% include icon.liquid id='info-circle' %} <b>Satisfactory data quality? </b></span><br>In principle you can now run all the rest of the pipeline at once. To do so you have set SRA and QC to "no" and to configure the other parts of `config_main.yaml`.
{: .ui.large.info.message}


## Trimming

If you put `TRIMMED: no`, there will be no trimming and the original FASTQ sequences will be mapped. 

If you put `TRIMMED: yes`, [Trim Galore](https://github.com/FelixKrueger/TrimGalore/blob/master/Docs/Trim_Galore_User_Guide.md) will remove low quality and very short reads, and cut the adapters. If you also want to remove a fixed number of bases in 5' or 3', you have to configure it. For instance if you want to remove the first 10 bases: 

```yaml
# ================== Configuration for trimming ==================

## Number of trimmed bases
## put "no" for TRIM3 and TRIM5 if you don't want to trim a fixed number of bases.
TRIM5: 10 #  integer or "no", remove N bp from the 5' end of reads. This may be useful if the qualities were very poor, or if there is some sort of unwanted bias at the 5' end. 
TRIM3: no # integer or "no", remove N bp from the 3' end of reads AFTER adapter/quality trimming has been performed. 
```

## Mapping

At this step you have to provide the path to your genome index as well as to a GTF annotation file and a BED file with CpG island coordinates. 

<span>{% include icon.liquid id='info-circle' %} <b>Use common banks!</b></span><br>Some reference files are shared between cluster users. Before downloading a new reference, check what is available at `/shared/bank/` (IFB) or `/shared/banks/` (iPOP-UP).
{: .ui.large.info.message}

```bash
[username@clust-slurm-client ~]$ tree -L 2 /shared/bank/homo_sapiens/
/shared/bank/homo_sapiens/
├── GRCh37
│   ├── bowtie2
│   ├── fasta
│   ├── gff
│   ├── star -> star-2.7.2b
│   ├── star-2.6
│   └── star-2.7.2b
├── GRCh38
│   ├── bwa
│   ├── fasta
│   ├── gff
│   ├── star -> star-2.6.1a
│   └── star-2.6.1a
├── hg19
│   ├── bowtie
│   ├── bowtie2
│   ├── bwa
│   ├── fasta
│   ├── gff
│   ├── hisat2
│   ├── picard
│   ├── star -> star-2.7.2b
│   ├── star-2.6
│   └── star-2.7.2b
└── hg38
    ├── bowtie2
    ├── fasta
    ├── star -> star-2.7.2b
    ├── star-2.6
    └── star-2.7.2b

30 directories, 0 files
```

If you don't find what you need, you can ask for it on [IFB](https://community.france-bioinformatique.fr/) or [iPOP-UP](https://discourse.rpbs.univ-paris-diderot.fr/c/ipop-up) community support. In case you don't have a quick answer, you can download (for instance [here](http://refgenomes.databio.org/)) or produce the indexes you need in your folder (and remove it when it's available in the common banks). Copy the path to the file you need and then paste the link to `wget`. When downloading is over, you might have to decompress the file. 

```
[username@clust-slurm-client Methylator]$ wget ???????
[username@clust-slurm-client index]$ tar -zxvf ???? 
```

- **GTF** files can be downloaded from [GenCode](https://www.gencodegenes.org/) (mouse and human), [ENSEMBL](https://www.ensembl.org/info/data/ftp/index.html), [NCBI](https://www.ncbi.nlm.nih.gov/assembly/) (RefSeq, help [here](https://www.ncbi.nlm.nih.gov/genome/doc/ftpfaq/#files)), ...
Similarly you can download them to the server using `wget`. 

- **CpG Islands** files can be downloaded from [UCSC goldenPath](https://hgdownload.soe.ucsc.edu/goldenPath/hg38/database/). For obtain the bed file, you can use this command (example with hg38): 

```sh
 wget -qO- http://hgdownload.cse.ucsc.edu/goldenpath/hg38/database/cpgIslandExt.txt.gz \
   | gunzip -c \
   | awk 'BEGIN{ OFS="\t"; }{ print $2, $3, $4, $5$6, $7, $8, $9, $10, $11, $12 }' \
   | sort-bed - \
   > cpgIslandExt.hg38.bed
```

<span>{% include icon.liquid id='info-circle' %} <b>Fill common banks!</b></span><br>Don't forget to give the links to the new references you made/downloaded to [IFB](https://community.france-bioinformatique.fr/) or to [iPOP-UP](https://discourse.rpbs.univ-paris-diderot.fr/c/ipop-up) support so that they can add them to the common banks.
{: .ui.info.message}

Be sure you give the right path to those files and adjust the other settings to your need: 


```yaml
# ================== Control of the workflow ==================
??????????

```

For an easy visualisation on a genome browser, BigWig files are generated. 



As at the moment the default project quota in 250 Go you might be exceeding the space you have (and may or may not get error messages...). So if the mapping fails, try removing files to get more space, or ask to increase your quota on [IFB Community support](https://community.cluster.france-bioinformatique.fr) or [iPOP-UP Community support](https://discourse.rpbs.univ-paris-diderot.fr/c/ipop-up). To see the space you have you can run:

``` 
[username@clust-slurm-client Methylator]$ du -h --max-depth=1 /shared/projects/YourProjectName/
```

and

```
[username@clust-slurm-client Methylator]$ lfsgetquota YourProjectName
```


##  Process BAM to BED (only for nanopore)

Si vous lancer le workflow avec des données nanopores la première étape consiste à convertir les fichiers BAM en BED à l'aide de l'outil modbam2bed conseillé par oxford nanopore technologie (ONT). Si vous avez déjà réalisé cette étape mettre à yes. If you already have BED files from modbam2bed you can put the path to the folder containing them. The BED should be named Sample_Merged.cpg.bed. 

```yaml
# # ===================== Configuration for process BAM files  ================== #
# =========================================================================== #

STARTFROMBED: no # put no if you start from BAM files
```


## Statistical analysis

A cette étape vous devez sélectionner les paramètres globaux de l'analysis statistique. 

```yaml
LEVEL: CpG # "CpG" or "Tiles" study by Tiles or by CpG 
TILESIZE: 250 
STEPSIZE: 1  # Tiles relative step size
NB_CPG_TILES: 1 # minimal number of CpG to keep a tile in the analysis
```
If Tiles was selected, define tile size, step size, and minimal number of CpG by tile. 
STEPSIZE correspond à la taille relative de l'écart entre deux tiles. Si il est à 1 les tiles sont non-chevauchantes , si il est à 0.5 les tiles se chevauchent à la moitiées ... 

### Exploratory analysis 

```yaml
# ===== Exploratory analysis ===== #
## params 
MINCOV: 5  # int, minimum coverage depth for the analysis
COV.PERC: 99.9 # to the coverage filter, choose the percentile for remove top ..% 
MINQUALI: 20  # int, minimum quality to keep a CPG for the analysis
DESTRAND: yes # combine information from F et R reads
UNITE: all # 'all' or 'one' (at least one per group)
```
COV.PERC correspond au pourcentage des CpG ou tiles les plus couvertes à conserver. Par exemple si COV.PERC = 99.9, seule les 1% les plus couvertes sont supprimées. Ce paramètres permet d'élliminer les biais du séquencage illumina dans les régions répétées (a revoir?). Attention, dans le cas du RRBS, mettre cette argument a 100 pour éviter de supprimer les régions volontairements enrichies. 

UNITE permet de choisir la manière dont vous souhaiter unifier vos réplicats. Par défaul (all), il est nécessaire qu'un CpG soit présent dans chacun des réplicats pour qu'il soit pris en considération. Sinon préciser le nombre minimale de fois qu'il doit apparaire pour être conservé. Par exemple si vous avez trois réplicats par condition et que vous sélectionné 2, il suffit qu'un CpG soit présent dans deux des trois réplicats pour qu'il soit conservé. 

### Differential methylation analysis

Finally you have to set the parameters for the differential methylation analysis. You have to define the comparisons you want to do (pairs of conditions). 

```yaml
COMPARISON : [["WT","1KO"], ["WT","DKO"], ["1KO","DKO"]] 
```
Et paramètrer votre analyse. 

```yaml
# ===== Differential analysis ===== #

DMR: yes # if yes, run DMR

## params (CPG or TILES)
LIST_SIGNIDIF: [10, 20, 25, 30] # SigDiffMeth en %, used in MKit_diff_bed.R
LIST_QVALUE : [0.001, 0.01, 0.05]  # used in MKit_diff_bed.R
QVALUE: 0.05  # QValue (used in Mkit_differential.Rmd)
SIGNIDIF: 10

## params (only DMR)
DMR_TYPE: regions  # regions or blocks, by default regions 
MIN_CPG: 5  # minimum number of CpGs to consider for a candidate DMR, by default 5, minimum 3
MAX_GAP: 1000 # maximum number of bp in between neighboring CpGs to be included in the same DMR, by default 1000
CUTOFF: 0.1 # cutoff of the single CpG methylation difference that is used to discover candidate DMR. by default 0.1
FDR: 0.05 # QVALUE for select significant DMR
```
LIST_SIGNIDIF et LIST_QVALUE sont utilisé pour la génération des bedgraphes. Pour chaque seuil de différences et pour chaque seuil de q-value un bedgraph est généré. 

Si DMR est YES, vous réalisez une analyse en DMR avec les mêmes pairs de comparaisons qu'en CpG ou Tiles. Attention, le package utilisé pour inféré les DMR a été conçu pour l'analyse WGBS. Si vous souhaitez réaliser une analyse en RRBS vous pouvez essayer de supprimer le smoothing, ou de modifier certains paramètres : ?? 
The default parameters are designed to focus on local DMRs (regions), generally in the range of hundreds to thousands of bp. If you choose blocks , the range increase on the order of hundreds of thousands to millions of bp. In this case, it's advised to decrease the cutoff


## Start the workflow

When the configuration files are fully adapted to your experimental set up and needs, you can **start the workflow** by running:

```
[username@clust-slurm-client Methylator]$ sbatch Workflow.sh wgbs
```