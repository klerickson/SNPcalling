# SNP calling
This is a tutorial for how to call Single Nucleotide Polymorphisms (SNPs) from UCE data after running [Phyluce Tutorial 1]([https://phyluce.readthedocs.io/en/latest/tutorial-one.html]). It outlines modifications that can be made to the scripts developed and used in Zaraz et. al. 2016 to extract SNPs from UCE reads.

It relies on the specific program versions that are installed on Harvey Mudd College's Purves server. 

Last updated June 16, 2022.  

## 1. Setup directory structure 

#### 1a. Create directory structure

    mkdir snp_calling
    cd snp_calling
    mkdir reads
    
#### 1b. For each sample you wish to call SNPs from, copy the cleaned reads from the phyluce tutorial one into your reads folder. 

	  cp -r /path/to/clean-fastq/sample-name /.../snp_calling/reads #change this line to match path to your reads, replace ... with sample name
  
## 2. Set a reference individual 

#### 2a. Choose reference individual by finding the sample with the most unique UCE contigs, which can be found by looking at the log files created when running phyluce.

    less /path/to/phyluce_assembly_match_contigs_to_probes.log #change path to match path to log file

#### 2b. Make a reference file named ``ref.conf`` which includes the following two lines
    
    cd snpcalling
    [ref] 
    #type your reference sample name here
    
#### 2c. Create a fasta of all the UCE loci found in your reference using phyluce programs

    phyluce_assembly_get_match_counts \
      	--locus-db /path/to/uce-search-results/probe.matches.sqlite \ #change this line to match path to your contigs
    	--taxon-list-config ref.conf \
    	--taxon-group ref \
    	--output ref-ONLY.conf
      
    phyluce_assembly_get_fastas_from_match_counts \
      --contigs /path/to/spades-assemblies/contigs \ #change this line to match path to your contigs
      --locus-db /path/to/uce-search-results/probe.matches.sqlite \ #change this line to match path to your probe matches
      --match-count-output ref-ONLY.conf \
      --output ref-ONLY-UCE.fasta 

## 3. Run the SNP calling script 

    cd snp_calling
    cp /data/mcfadden/kerickson/Programs/snpcalling.sh /.
    nohup bash snpcalling.sh > snpcalling.log

## 4. Filter SNPs using vcftools

The following parameters are useful for our the SNP data sets we create:
<ul>
<li> <code>--min-alleles 2</code> and <code>--max-alleles 2</code> ensures that we only included biallelic sites </li>
<li> <code>--thin 1000</code> ensures that no two snps are within a 1000 bp from one another so that we only choose 1 SNP per UCE locus </li>
<li> <code>-- max-missing</code>  determines the amount of missing data that is allowed and can be changed to balance yield and completeness in our data set (e.g. <code>--max-missing 0.95</code> means 95% taxon completeness) </li>
<li> <code>--max-non-ref-af 0.99</code> paraments ensures that no SNPs that are the same across all samples are included </li>
  <li> <code>--recode</code> creates a <code>.vcf</code> file which we use for downstream analysis </li>
</ul>

Here is how the above filters can be run: 

    vcftools --vcf genotyped_X_samples_only_PASS_snp_5th.vcf --min-alleles 2 --max-alleles 2 --thin 1000 --max-missing 0.95 --max-non-ref-af 0.99 --recode --out filtered_vcf95
    
``vcftools`` can be found at ``/data/mcfadden/aquattrini/PROGRAMS/vcftools/bin/vcftools``

More parameters can be added to filter SNPs, subset the individuals, produce more output files and more. Documentation for available parameters can be found on the [VCTools Webpage]([https://duckduckgo.com](http://vcftools.sourceforge.net/man_latest.html))

## 5. Convert filtered snps into a .str file for downstream analysis
		
    phyluce_snp_convert_vcf_to_structure --input filtered_vcf95.recode.vcf --output input_structure.str

## 6. Next Steps
Your final ``.str`` file can be used to run STRUCTURE on purves and can be read into R and analyzed using a variety of programs (such as DAPC and Unsupervised Random Foresting analyses). 

# Running STRUCTURE
## 1. Make sure .str file does not have a header line of loci labels. If it does, you can remove it using:
	nano filtered_snps.str
	command+K to remove first line of text 
	
## 2. Make folder to run structure in, and copy the following files into folder:
	/data/mcfadden/kerickson/STRUCTURE/input.py 
	/data/mcfadden/kerickson/STRUCTURE/strauto_1.py 
	/data/mcfadden/kerickson/STRUCTURE/structure 
	/data/mcfadden/kerickson/STRUCTURE/parallel
	/path/to/filtered_snps.str
## 3. Edit input.py
In #1, change ``maxpops = `` to be the largest K value you would like to run
In #4, change ``dataset = `` to be the name of your .str file
In #6, change ``numind =`` to be the number of individuals in your analyses 
In #7, change ``numloci =`` to be the number of loci
(the number of individuals and loci in your file can be found in the .log file created when you used .vcf to filter your snps) 

## 4. Create binary file to run structure with your input by running the following:
	python2.7 strauto_1.py
It will prompt you to review and accept the parameters you set in input.py

## 5. Edit extraparams
	nano extraparams
	Change ???LOCISPOP = 1??? to ???LOCISPOP = 0???

## 6. Run the program
	nohup ./runstructure
	
## 7. Visualize results
Results for each K will be populated in the folder ``results_f``. These files can be read into R and made into plots using commands in the R library ``pophelper``

