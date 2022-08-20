# Variant Calling Pipeline using GATK4
Published by Mohammed Khalfan on 2020-03-25 [Variant Calling](https://gencore.bio.nyu.edu/variant-calling-pipeline-gatk4/).<br>

This is an updated version of the variant calling pipeline post published in 2016 (link). This updated version employs GATK4 and is available as a containerized Nextflow script on GitHub.

Identifying genomic variants, including single nucleotide polymorphisms (SNPs) and DNA insertions and deletions (indels), from next generation sequencing data is an important part of scientific discovery. At the NYU Center for Genomics and Systems Biology (CGSB) this task is central to many research programs. For example, the Carlton Lab analyzes SNP data to study population genetics of the malaria parasites Plasmodium falciparum and Plasmodium vivax. The Gresham Lab uses SNP and indel data to study adaptive evolution in yeast, and the Lupoli Lab in the Department of Chemistry uses variant analysis to study antibiotic resistance in E. coli. 

To facilitate this research, a bioinformatics pipeline has been developed to enable researchers to accurately and rapidly identify, and annotate, sequence variants. The pipeline employs the Genome Analysis Toolkit 4 (GATK4) to perform variant calling and is based on the best practices for variant discovery analysis outlined by the Broad Institute. Once SNPs have been identified, SnpEff is used to annotate, and predict, variant effects. 
