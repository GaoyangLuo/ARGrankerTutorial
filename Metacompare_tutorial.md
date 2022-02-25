# Metacompare Tutorial

Create an conda environment for metacompare
```sh
conda create -n metacompare -y
conda activate metacomre
conda install -c bioconda blast
conda install git
conda install python=3.9
pip install pandas 
pip install biopython
```
Change a directory where you want to store metacompare. Here I chose the directory "software" to install.
```sh
git clone https://github.com/minoh0201/MetaCompare
```
Make directory BlastDB under MetaCompare directory and change the woring directory to it.
```sh
cd software/Metacompare && mkdir blastdb && cd blastdb
```
Download the compressed Blast Database file from the web server (25 GB) and uncompress it.
```sh
wget -b -c http://bench.cs.vt.edu/ftp/data/metacomp/BlastDB.tar.gz --no-check-certificate
```


