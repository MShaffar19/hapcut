HAPCUT, version 0.7, Vikas Bansal (vbansal@scripps.edu), last modified: March 25 2014

## Introduction

HapCUT is a max-cut based algorithm for haplotype assembly that uses the mix of sequenced fragments from the two chromosomes of an individual, this program can be applied to sequence data generated from next-generation sequencing platforms. HapCUT takes as input the aligned SAM/BAM files for an individual diploid genome and the list of variants, and outputs the phased haplotype blocks that can be assembled from the sequence reads. HapCUT is described in the following artice:

An efficient and accurate algorithm for the haplotype assembly problem Bioinformatics. 2008 Aug 15;24(16):i153-9.


## Requirements:

To run HapCUT, you need at least the following files: 

1. A file with aligned reads for the individual diploid genome. HapCUT currently accepts BAM files that are coordinate sorted, i.e. by chromosomal position. 

2. A file with the list of variants and genotypes for the diploid genome. HapCUT currently accepts variants file in VCF format (variants and genotypes for a single individual). For specification of the VCF format, please see http://www.1000genomes.org/wiki/Analysis/Variant%20Call%20Format/vcf-variant-call-format-version-41. Note that VCF format is required for phasing indels. 

3. The reference sequence file used to align the reads. This file should be in the fasta format and indexed using samtools to generate a .fai file. This file is optional but required for phasing indels. 


## Format of input and output files:

1. BAM file with read alignments: This file should be coordinate sorted, e.g. using the "samtools sort" command. In addition, the mate-chromosome and mate-position (columns 7 and 8 for an aligned read) should reflect the reference chromosome and position of the mate for paired-end reads. 


2. VCF file: This file should contain the variants and genotypes for a single individual that is to be phased. Please make sure that the VCF file conforms to the standard VCF format, since HapCUT does not check this. 

3. hapCUT variant file: Each line of this file represents a heterozygous variant (SNP/indel). The file should be sorted with respect to chromosomal position. Format of each line is: variantid chromosome position allele1 allele2 genotype genotype_quality. An example of a variant in this file is as follows: 

SNP chr4 45322434 A C A/C 65  (he chromosomal position is 1-based rather than 0-based) 

It is preferrable to use the VCF format for variants. 

4. Output file: this file contains the phased haplotype blocks generated by HapCUT. Each block (connected component in the fragment matrix) has the following format: 

"BLOCK: offset:" first_variant_block "len:" length_of_block "phased": phased_variants_block SPAN: lengthspanned MECscore score fragments #fragments

Subsequently, each line represents a variant that is phased as part of the block in the following format:

variant_id haplotype_1 haplotype_2 chromosome position refallele variantallele genotype allele_counts:genotype_likelihoods:delta:MEC_variant 

	Haplotype_1 and Haplotype_2 correspond to the two phased haplotypes (0 corresponds to the refallele and 1 corresponds to the variantallele)

	allele_counts: two integers that correspond to the number of reads supporting the two haplotypes at this variant

	genotype_likelihoods: likelihoods for three possible diploid genotypes (0/0, 0/1, 1/1) for this variant using the haplotype informative reads and conditional on the current haplotypes

	delta: maximum of the two values (GL(0/0)-GL(0/1), GL(1/1)-GL(0/1)), positive value of delta indicates that this variant could potentially not be heterozygous. 

	MEC_variant: reduction in the MEC score of the current assembled haplotypes by removing this variant. High value of MEC_variant indicates that the variant is either not a real variant or that the variant is not heterozygous. Variants for which delta >=3 and MEC_variant is >=2 are flagged as "FV". 

The output of HapCut can be combined with the original input VCF to produce a phased VCF using [fgbio's HapCutToVcf tool](https://github.com/fulcrumgenomics/fgbio).


Running the program:
=====================

The following two steps are required for assembling haplotypes for an individual genome:

1. extract haplotype-informative reads (reads that cover at least two heterozygous variants) from the aligned bam file

				./extract_hairs --VCF variantfile.VCF --bam alignedreads.sorted.bam --maxIS 600 > fragment_matrix_file

   For extracting reads covering both SNPs and indels, the reference file (in fasta file and indexed using samtools) is required: 

  				./extract_hairs --VCF variantfile.VCF --bam alignedreads.sorted.bam --maxIS 600 --ref reference.fasta --indels 1 > fragment_matrix_file 


2. Run HapCUT to assemble haplotypes from the haplotype informative reads:

   				./HAPCUT --fragments fragment_matrix_file --VCF variantfile.VCF --output output_haplotype_file --maxiter maxiter  > hapcut.log

The default number of iterations for HAPCUT is 100. As the program iterates, it will output the current best haplotype assembly to the file output_haplotype_file after every 10 iterations.  


HapCUT input arguments:

	 1. --fragments: list of fragments that cover at least two heterozygous variants (can be unsorted)
	 2. --VCF: VCF variant file with genotypes for a single individual, same as that used for parsing SAM/BAM file for generating fragments file
	 3. --output: output file for haplotype phasing
	 4. --maxiter: number of iterations to run HAPCUT


How to phase fosmid pooled sequencing data (Kitzman et al. Nat. Biotech. 2011) 
======================================================================================================================================================================

1. Align reads for each pool to reference genome using your favorite aligner.
2. Use 'samtools targetcut' command to generate consensus sequences for each long fragment/fosmid 
3. run 'extracthairs' to generate variant calls at heterozygous sites in each fragment (this should be done for each pool separately)
4. combine the fragment files from different pools into a single fragment file
5. run HAPCUT using this fragment file and VCF variant file using options "--fosmids 1" and "--sf 1" 

In the future, we plan to release a program to extract fragment sequences from the bam files for each pool that will improve the accuracy of the haplotyping. This approach has been successfully tested on data from a Gujarati Indian individual (NA20847) generated by Kitzman et al. Nat. Biotech 2011. 


Revision history
==================

March 25 2014

	- program can handle heterozygous sites for which both alleles are different from reference (genotype = 1/2 in VCF file) 
	- extract_hairs can process PacBio reads with or without the quality values
	- if the BAM file has no quality values, use --minq <INT> where <INT> reflects the average accuracy of the reads (for an error rate of 10%, use a value of 10)  
	- extract_hairs can handle the '=' CIGAR operation 
	- for long reads, added "--longreads" option to reduce memory requirements for storing the fragment graph
	

Sept 11 2013 

	- VCF format is now the only format for specifying variants
	- extract_hairs allows large insert sizes (10-100 megbases) for phasing using mate-pair ligation sequencing data
	- maximum number of iterations for finding bestcut for each component is limited to 100, can be increased for marginally better performance
	- added support for phasing fosmid pooled sequencing data (see Kitzman et al. Nat. Biotech. 2011). Use options "--fosmids 1" and "--sf 1" for phasing fosmid data. 

July 04 2012 
	
	- VCF input file is recommended, limited support for old variant file format
	- SAM format is not supported anymore 
	- bugs fixed in extract_hairs 
	- for each connected component, hapCUT only outputs variants that are connected (variants that are spanned are not output)
	
May 04 2012

	-bug in hapcut due to unchecked condition in max-cut computation

May 01 2012  

	- handles overlapping paired-end reads properly 
	- improvements in HapCUT resulting in increased speed

April 19 2012

	- supports BAM files and VCF files for input
	- fixed bug related to duplicate reads in extracthairs program
	- added support for indels (preliminary)
	- print information about #reads supporting two alleles, genotype likelihoods for each variant in output file


