_IN PROGRESS_

# zebrafish_orthologs
Automated method to find orthologs of human genes in zebrafish.

This is a quick and easy method to find orthologs of human genes in zebrafish based on the working definition of Reciprocal Best Hit (RBH).
The method and scripts are originally from [Nicolas Schmelling] (https://github.com/schmelling). I only tweaked them for my application.

The bioRxiv preprint he used this method for: https://www.biorxiv.org/content/early/2017/02/14/075291
His protocols.io explaining the method and Terminal commands: https://www.protocols.io/view/reciprocal-best-hit-blast-q3rdym6
His GitHub with the original Python scripts and more explanations: https://github.com/schmelling/reciprocal_BLAST

 `mkdir project`
 `cd project`
 
 `pip3 biopython`

# 1. Download the Homo sapiens sequences you are interested in.
i.e. amino acids sequences in fasta format from NCBI/protein.
* One fasta file per sequence.
* Try to get RefSeq sequences.
* If possible, avoid taking an isoform. For example the preprotein should be a good choice. 
* If not possible, try taking the longest possible isoform while avoiding the X isoforms (eg. isoform X11) as these are computationally predicted.

# 2. Download Danio rerio proteome
`awk -F '\t' '{if($12=="Chromosome") print $20}' assembly_summary.txt > assembly_summary_chromosome.txt`

`mkdir RefSeqChromosomeGenomes
mkdir RefSeqChromosomeReports
mkdir db`

`for next in $(cat assembly_summary_chromosome.txt);
do
wget -P RefSeqChromosomeGenomes "$next"/*protein.faa.gz;
wget -P RefSeqChromosomeReports "$next"/*assembly_report.txt;
done`

`gunzip RefSeqChromosomeGenomes/*.gz`

`grep -c "^>" GCF_000002035.6_GRCz11_protein.faa`
52,829 protein sequences as of October 2018.

# 3. Create BLAST database
`cd db`
`makeblastdb -in all_genomes.fasta -dbtype 'prot' -out zebraprots`

# 4. Run the forward BLAST
i.e. which proteins in the zebrafish proteome look like my human proteins?
`mkdir seq`
`cd seq`

`for f in seq/*.fasta
do
blastp -query "$f" -db zebraprots -out "${f%.fasta}_blast.xml" -outfmt 5 -evalue 10e-20 -word_size 6 -num_alignments 100 -num_threads 8
done`
NB: Parameters can be changed in the BLAST command: E-value, word size, number of output alignments, number of threads to use.
I have used the default BLAST parameters.

# 5. Parse the forward hits into fasta files
pyton parse_hits.py

# 6. Run the reciprocal BLAST
i.e. BLAST the forward hits in zebrafish back in the human proteome. 
If one zebrafish protein gives you back the original human protein, it is an ortholog based on the definition of the Reciprocal Best Hits.
