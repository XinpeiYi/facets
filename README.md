# FACETS
**Please read [the manuscript](https://academic.oup.com/nar/article/44/16/e131/2460163) and the [userguide](https://github.com/mskcc/facets/blob/master/vignettes/FACETS.pdf) before using**.  

Github: [mskcc/facets](https://github.com/mskcc/facets)  
DOI: [https://doi.org/10.1093/nar/gkw520](https://academic.oup.com/nar/article/44/16/e131/2460163)  
Citation: [Ronglai Shen, Venkatraman E. Seshan; FACETS: allele-specific copy number and clonal heterogeneity analysis tool for high-throughput DNA sequencing, Nucleic Acids Research, Volume 44, Issue 16, 19 September 2016, Pages e131, https://doi.org/10.1093/nar/gkw520](https://academic.oup.com/nar/article/44/16/e131/2460163)

This repository contains an implementation of [FACETS](https://github.com/mskcc/facets), an algorithm to estimate fraction and allele specific copy number from tumor/normal sequencing by Ronglai Shen and Venkatraman E. Seshan from Memorial Sloan-Kettering Cancer Center's Department of Epidemiology and Biostatistics, for FireCloud. FACETS is run on tumor-normal pairs to determine the purity and ploidy of a tumor sample, as well as infer allele-specific copy number.

To best use this repository, please:
- Read [the manuscript](https://academic.oup.com/nar/article/44/16/e131/2460163) and the [userguide](https://github.com/mskcc/facets/blob/master/vignettes/FACETS.pdf)
- Consider the coverage of your samples, especially when thinking of the ndepth (minimum read depth in normal) parameter. 
- Check the emflags to see if your sample is noisy
- See how stable your called solution(s) are with the .facets_iterations.txt and pdf outputs
- Observing NA purity values? [See this Github issue](https://github.com/mskcc/facets/issues/68) and also [this one](https://github.com/mskcc/facets/issues/66)
- Read and search the [issues, both open and closed, on Github](https://github.com/mskcc/facets/issues?utf8=%E2%9C%93&q=) to learn more about the method
- Read the help text on each function called in facets.R
- Inspect each sample closely

Docker image: [vanallenlab/facets](https://hub.docker.com/r/vanallenlab/facets/)  
FireCloud method: [vanallenlab/facets](https://portal.firecloud.org/#methods/vanallenlab/facets/)

## Task details

### Pileup

#### Install snp-pileup
- Install HTSlib : http://www.htslib.org/download/
- tar -xf htslib-1.11.tar.bz2
- cd htslib-1.11
- ./configure --prefix=/data/xyi/software/htslib-1.11
- make
- make install
- g++ -std=c++11 -I /data/xyi/software/htslib-1.11/include snp-pileup.cpp -L /data/xyi/software/htslib-1.11/lib -lhts -Wl,-rpath=/data/xyi/software/htslib-1.11/lib -o snp-pileup

#### Run pileup

- Download dbSNP hg38 databse : wget https://ftp.ncbi.nih.gov/snp/organisms/human_9606_b151_GRCh38p7/VCF/common_all_20180418.vcf.gz
- /data/xyi/snp-pileup common_all_20180418.vcf -q15 -Q20 -P100 -r100,0 111_112.100.facets 111.sorted.bam 112.sorted.bam

Read depth and allele counts at sites of common single nucleotide variant are observed in both the tumor and normal bams based on a provided VCF. 

Inputs:
- Pair name
- Normal bam and corresponding index
- Tumor bam and corresponding index
- VCF and corresponding index for common variants
- Minimum mapping quality
- Minimum based quality
- Minimum read depth in normal and tumor
- Number of pseudo SNPs

Read the [pileup documentation](https://github.com/mskcc/facets/tree/master/inst/extcode) for more details of each input. The default VCF of common variants in the FireCloud method configuration is [human 9606 b151 GRCh37](ftp://ftp.ncbi.nlm.nih.gov/snp/organisms/human_9606_b151_GRCh37p13/VCF/), if you do not have access please download the appropriate VCF and host in a bucket that you have access to.

Outputs:
- Pileup in gunzip format. Read more in the [userguide](https://github.com/mskcc/facets/blob/master/vignettes/FACETS.pdf).

[Relevant codeblock in WDL](https://github.com/vanallenlab/facets/blob/lab_harmonize/facets.wdl#L82-L125)

### FACETS
Estimates fraction and allele specific copy number from paired tumor/normal sequencing. As a result, also estimates purity and ploidy of a given tumor sample. This implementation will run FACETS 10 times across different seeds to observe how stable the inferred purity and ploidy values are for a given pair, the seed with a purity closest to the median purity value is selected. If you would prefer to not iterate and use a specific seed, set `seed_iterations` to `1` and specify your preferred seed with `seed_initial`.

Inputs:
- Pair name
- Pileup
- Minimum read depth in normal
- Critical value (cval), the threshold for determining if a change exists
- Maximum number of expectation maximization iterations (maxiter)
- Initial seed to set R to
- Number of seeds to test

Outputs:
- Genome segments plot, displaying copy and log ratios across chromosomes
- Diagnostics plot, displaying the segment summaries
- Iterations plot, displys ploidy and purity for all iterations performed across seeds
- Copy number cellular fraction, total and minor allele counts for all segments
- Estimated purity and ploidy
- Flags and warnings generated by FACETS
- Seed value closest to median purity
- Number of seeds that resulted in purity values equaling NA

[Relevant codeblock in WDL](https://github.com/vanallenlab/facets/blob/lab_harmonize/facets.wdl#L127-L167)

### Infer Whole-genome doubling
Infers whole genome doubling based on [Bielski CM, Zehir A, Penson AV, et al. Genome doubling shapes the evolution and prognosis of advanced cancers](https://doi.org/10.1038/s41588-018-0165-1). Calculates major copy number (MCN) estimate based on total copy number (TCN) estimate and minor copy number (LCN) estimate from FACETS and calls whole-genome doubling if the average MCN across the autosomal genome is greater than 2. In cases that LCN is equal to NA, a value of 0 is used.

Inputs:
- Copy number cellular fractions

Outputs:
- Fraction major copy number greater than 2
- Putative whole genome doubling, if the fraction of MCN across the autosomal genome is greater than 0.5

[Relevant codeblock in WDL](https://github.com/vanallenlab/facets/blob/lab_harmonize/facets.wdl#L169-L191)

### Infer precent genome altered (PGA)
Infers the percentage of the genome altered by copy number alterations (differences from normal ploidy). Commonly used in a variety of papers, ([example here](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6145837/)). This is done by taking all the segments that do not match diploid copy number (2 for autosomes and 1 for sex chromosomes) and computing their sizes, then dividing by the total size of all the segments. In cases that total copy number is equal to NA, a value of 0 is used, although this should be impossible.

Inputs:
- Copy number cellular fractions

Outputs:
- Fraction of genome altered

### Additional reading:
- [Ronglai Shen, Venkatraman E. Seshan; FACETS: allele-specific copy number and clonal heterogeneity analysis tool for high-throughput DNA sequencing, Nucleic Acids Research, Volume 44, Issue 16, 19 September 2016, Pages e131, https://doi.org/10.1093/nar/gkw520](https://academic.oup.com/nar/article/44/16/e131/2460163)
- [Bielski CM, Zehir A, Penson AV, et al. Genome doubling shapes the evolution and prognosis of advanced cancers](https://doi.org/10.1038/s41588-018-0165-1)
- [FACETS Github issue, "filtering and interpreting results"](https://github.com/mskcc/facets/issues/62)
- [FACETS Github issue, "About parameters setting for targeted panel](https://github.com/mskcc/facets/issues/81)
- [FACETS Github issue, "About snp pileup, psuedo snps](https://github.com/mskcc/facets/issues/61)
