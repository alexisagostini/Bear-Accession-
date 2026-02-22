# start the bear project 

## collect the information

### Reference Genome

On ncbi I dwnl the genome an the annotation
```bash
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/023/065/955/GCF_023065955.2_UrsArc2.0/GCF_023065955.2_UrsArc2.0_genomic.fna.gz
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/023/065/955/GCF_023065955.2_UrsArc2.0/GCF_023065955.2_UrsArc2.0_genomic.gff.gz
gunzip GCF_023065955.2_UrsArc2.0_genomic.fna.gz
gunzip GCF_023065955.2_UrsArc2.0_genomic.gff.gz
mv GCF_023065955.2_UrsArc2.0_genomic.fna.gz REF_GENOME.fasta
mv GCF_023065955.2_UrsArc2.0_genomic.gff.gz Annotation_RDF_genome.gff
```




## Stating the pipeline 

as previous the previous didn't change the most exept that I add option of the genome's annotation 
