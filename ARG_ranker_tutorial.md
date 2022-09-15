# ARG_ranker Tutorial by Gaoyang
---
## Preset relevant software
Download relevant software.
```sh
#run the code below by line
#pre-set conda environment
export PATH="~/miniconda/bin:$PATH"
source ~/.bashrc
conda --help
conda create -n argranker -c bioconda diamond blast kraken2 python=3.9
#check programme
diamond --help
kraken2 --help
python
exit()
#pre-set QC pipeline
conda install fastqc -y
conda install trimmomatic -y
```

## **Build the Kraken2 database**
A Kraken 2 database is a directory containing at least 3 files:
- hash.k2d: Contains the minimizer to taxon mappings
- opts.k2d: Contains information about the options used to build the database
- taxo.k2d: Contains taxonomy information used to build the database

Before doing the database construction, you should fix some problems. If you do not do the following step, you will find problems like:
```sh
rsync_from_ncbi.pl: unexpected FTP path (new server?) for https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/762/265/GCF_000762265.1_ASM762
```
Fixing the problem

Find the file below
`$ ~/miniconda3/envs/argranker/libexec/`

You can copy the code below:
```sh
cd ~/miniconda3/envs/argranker/libexec
vi rsync_from_ncbi.pl
```
Then change the `ftp` to `https`, like this:
```sh
#the origin one
if (! ($full_path =~ s#^ftp://${qm_server}${qm_server_path}/##)) 
#the new one
if (! ($full_path =~ s#^https://${qm_server}${qm_server_path}/##)) 
```
After fixing all the problem, you can copy the code below:
```sh 
#activate the conda environment
export PATH="/lomi_home/gaoyang/miniconda/bin:$PATH"
source ~/.bashrc
#activate argranker environment
source activate argranker
DBNAME=~/db/kraken2/202203
mkdir -p $DBNAME
kraken2-build --standard --threads 2 --db $DBNAME
```
The complete database should includes all the files below:
```sh
hash.k2d
library
opts.k2d
seqid2taxid.map
taxo.k2d
taxonomy
unmapped.txt
```
**Take notifications**
- while running `kraken2-build --standard --threads 2 --db $DBNAME`, don't use the `nohup` or `&` commond line.
- when you see `mv: try to overwrite ‘assembly_summary.txt’, overriding mode 0444 (r--r--r--)`, type `y` and enter.
- If you run the commond on management node, please use 2 threads to prevent node congestion.
- The file lack problem can be fixed by the following commonds:`kraken2-build --build --threads 2 --db standard_db`, and please notice that do not use more than the manxium threads to run, for example, if the maximum threads are 20, but you run it with 24 threads, such as `kraken2-build --build --threads 24 --db $DBNAME`, you will confront with errors like this:
```sh
build_db: OMP only wants you to use 20 threads
xargs: cat: terminated by signal 13
```
The default library includes archaea、bacteria、human、plasmid、UniVec_Core、viral. You can download other libraries like protozoa, fungi or plant like:
```sh
kraken2-build --download-library fungi --db $DBNAME
kraken2-build --download-library protozoa --db $DBNAME
kraken2-build --download-library plant --db $DBNAME
```
Useing 20 threads to complete all the database construction, you will see results like:
```sh
$ kraken2-build --build --threads 20 --db $DBNAME
Creating sequence ID to taxonomy ID map (step 1)...
Sequence ID to taxonomy ID map already present, skipping map creation.
Estimating required capacity (step 2)...
Estimated hash table requirement: 59103177872 bytes
Capacity estimation complete. [17m25.833s]
Building database files (step 3)...
Taxonomy parsed and converted.
CHT created with 16 bits reserved for taxid.
Completed processing of 126460 sequences, 127523632805 bp
Writing data to disk...  complete.
Database files completed. [7h17m57.509s]
Database construction complete. [Total: 7h35m23.480s]
```
To caculate ARGs relative abundance by using  copy number of ARGs per 16S, we should download the 16S database:
```sh
kraken2-build --db ~/db/kraken2/202203 --special greengenes --threads 4
```
---

### QC
trimming
```sh
 

source /lomi_home/gaoyang/miniconda/bin/activate argranker

#fqc
/lomi_home/gaoyang/miniconda/envs/argranker/bin/fastqc ./*fastq -t 20 -o /lomi_home/gaoyang/microplastic_test/aus_ERP124866/workdir/qc_control

#SingleEnd
for filename in *.fastq
do
base=$(basename $filename .fastq)
python /lomi_home/gaoyang/miniconda/envs/argranker/share/trimmomatic//trimmomatic SE -threads 6 -phred33 ${base}.fastq ${base}_clean.fq ILLUMINACLIP:/lomi_home/gaoyang/miniconda/envs/argranker/share/trimmomatic/adapters/TruSeq3-SE.fa:2:40:15 LEADING:2 TRAILING:2 SLIDINGWINDOW:4:20 MINLEN:25
done

#PairEnd
for filename in *.man_1.fastq
do
base=$(basename $filename .man_1.fastq)

reads_1=${base}.man_1.fastq
reads_2=${base}.man_2.fastq

python /lomi_home/gaoyang/miniconda/envs/argranker/share/trimmomatic//trimmomatic PE -threads 4 -phred33 ${reads_1} ${reads_2} ${base}_clean_1.fq ${base}_u1 ${base}_clean_2.fq ${base}_u2 \
ILLUMINACLIP:/lomi_home/gaoyang/miniconda/envs/argranker/share/trimmomatic/adapters/TruSeq3-PE.fa:2:40:15 \
LEADING:2 TRAILING:2 \
SLIDINGWINDOW:4:20 MINLEN:25
done

#fqc_again
/lomi_home/gaoyang/miniconda/envs/argranker/bin/fastqc ./*fastq -t 4 -o /lomi_home/gaoyang/st_test/AUS_ERP124866/workdir/qc_control
```

### Classify_ARGs
After clean reads preparation, we can run arg_ranker to classify ARGs.
```sh
#creat a directory to resotry all the soft link of .fq files
KKDB=/lomi_home/gaoyang/db/kraken2/202203
mkdir run && cd $_
dir=`pwd`
ln -s /lomi_home/gaoyang/microplastic_test/aus_ERP124866/1_clean/* .
#$input
input=${dir}
#activate_environment
source /lomi_home/gaoyang/miniconda/bin/activate argranker
#start_running
arg_ranker -i ${input} -t 4 -kkdb $KKDB
#follow_the_tips
chomd +x $(dirname `pwd`)/arg_ranking/script_output/arg_ranker.sh
#run_arg_ranker.sh
sh $(dirname `pwd`)/arg_ranking/script_output//arg_ranker.sh
```


