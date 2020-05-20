_IN PROGRESS_

# Zebrafish orthologs
Semi-automated method to find orthologs of human proteins in zebrafish proteome.

This is a quick and easy method to find orthologs of human genes in zebrafish based on the working definition of Reciprocal Best Hit (RBH).
The method and scripts are originally from [Nicolas Schmelling](https://github.com/schmelling). I only tweaked them for my application.

The bioRxiv preprint he used this method for: https://www.biorxiv.org/content/early/2017/02/14/075291

His protocols.io explaining the method and Terminal commands: https://www.protocols.io/view/reciprocal-best-hit-blast-q3rdym6

His GitHub with the original Python scripts and more explanations: https://github.com/schmelling/reciprocal_BLAST

# 0. Folders and dependencies
You probably want to do the next steps in a folder for your project:

 ```bash
 mkdir xxx
 cd xxx
 ````
 
 You will need Biopython. Install it with eg.
 
 ```bash
 pip3 biopython
 ```

# 1. Download your _Homo sapiens_ sequences of interest.
i.e. amino acids sequences in fasta format from NCBI/protein.
* One fasta file per sequence.
* Try to get RefSeq sequences.
* If possible, avoid taking an isoform. For example, the preprotein should be a good choice. 
* If not possible, try taking the longest possible isoform while avoiding the X isoforms (eg. isoform X11) as these are computationally predicted.

Create a folder `seq` and move your fasta files there.

# 2. Download _Danio rerio_ proteome

```bash
wget ftp://ftp.ncbi.nih.gov/genomes/refseq/vertebrate_other/Danio_rerio/reference/GCF_000002035.6_GRCz11/GCF_000002035.6_GRCz11_protein.faa.gz/
```

```bash
gunzip GCF_000002035.6_GRCz11_protein.faa.gz
```
How many sequences are there?
```bash
grep -c "^>" GCF_000002035.6_GRCz11_protein.faa
```
52,829 protein sequences as of October 2018.

```bash
python change_fasta_header.py
```

# 3. Create BLAST database

```bash
cd db
```

```bash
makeblastdb -in all_genomes.fasta -dbtype 'prot' -out zebraprots
```

`zebraprots` is the name of the database, you can change it.

# 4. Run the forward BLAST
i.e. which proteins in the zebrafish proteome align to my human proteins?

```
for f in seq/*.fasta
do
blastp -query "$f" -db zebraprots -out "${f%.fasta}_blast.xml" -outfmt 5 -evalue 10e-20 -word_size 6 -num_alignments 100 -num_threads 8
done
```

NB: Parameters can be changed in the BLAST command: E-value (`evalue`), word size (`word_size`), maximum number of output alignments (`num_alignments`), number of threads to use (`num_threads`).

I have used the default BLAST parameters.

# 5. Parse the forward hits into fasta files
i.e. creates for each human protein a fasta file containing the zebrafish proteins it aligns to.

```bash
python parse_hits.py
```

# 6. Run the reverse BLAST
i.e. BLAST the forward hits in zebrafish back in the human proteome. 
If one zebrafish protein gives you back the original human protein, it is an ortholog based on the definition of the Reciprocal Best Hits.

Download the human proteome.

```
wget ftp://ftp.ncbi.nih.gov/genomes/refseq/vertebrate_mammalian/Homo_sapiens/reference/GCF_000001405.38_GRCh38.p12/GCF_000001405.38_GRCh38.p12_protein.faa.gz/
```

```bash
gunzip GCF_000001405.38_GRCh38.p12_protein.faa.gz
```

How many sequences are there?

```bash
grep -c "^>" GCF_000001405.38_GRCh38.p12_protein.faa
```
113,620 protein sequences as of October 2018.

```bash
python change_fasta_header.py
```

Make a database searchable by BLAST:
```
makeblastdb -in GCF_000001405.38_GRCh38.p12_protein.faa -dbtype 'prot' -out humanprots.fasta
```

`humanprots` is the name of the database, you can change it.

Run the reverse BLAST
```bash
for f in db/seq/*_matches.fasta
do
blastp -query "$f" -db db2/humanprots -out "${f%_matches.fasta}_back_blast.xml" -outfmt 5 -word_size 6  -num_alignments 1
done
```

# 7. Manually look at the potential orthologs
I unfortunately could not find a solution to automate this step, this is mainly because one protein can have different names or different isoforms.

Look at the results with:

```bash
python hits_to_csv_test.py
```

For each human protein of interest, I change in `hits_to_csv_test.py` the file name.

Look at the names of the human proteins for your protein of interest. It could be an isoform or more rarely a synonym/previous name to the one you are used to.

Each time, copy/paste the zebrafish hit into NCBI to look at the record.
Usually, they will all be isoforms of the same protein but I have had cases where there seems to be two orthologs for one human protein.

# 8. Match your ortholog with the ZFIN record

It is not always clear which ZFIN record matches your ortholog's NCBI record.
For instance, if there are two copies of your zebrafish ortholog (`a` and `b`), NCBI does not seem to mention which one it is.
If it is unclear, the safest thing to do is to get the ZFIN protein sequence (`Sequence information` > `UniProt...`) and align it to the NCBI sequence with a Protein BLAST (tick `Align two or more sequences`).
You expect > 99% identity if they are the same.
