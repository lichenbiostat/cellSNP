=======
cellSNP
=======

|PyPI| |Build Status| |DOI|

.. |PyPI| image:: https://img.shields.io/pypi/v/cellSNP.svg
    :target: https://pypi.org/project/cellSNP
.. |Build Status| image:: https://travis-ci.org/PMBio/cellSNP.svg?branch=master
   :target: https://travis-ci.org/PMBio/cellSNP
.. |DOI| image:: https://zenodo.org/badge/145724973.svg
   :target: https://zenodo.org/badge/latestdoi/145724973

cellSNP aims to pileup the expressed alleles in single-cell or bulk RNA-seq 
data, which can be directly used for donor deconvolution in multiplexed 
single-cell RNA-seq data, particularly with vireo_, which assigns cells to 
donors and detects doublets, even without genotyping reference.

cellSNP heavily depends on pysam_, a Python interface for samtools and bcftools. 
This program should give very similar results as samtools/bcftools mpileup. 
Also, there are two major differences comparing to bcftools mpileup:

1. cellSNP can pileup either the whole genome or a list of positions, with 
   directly splitting into a list of cell barcodes, e.g., for 10x genome. With 
   bcftools, you may need to manipulate the RG tag in the bam file if you want 
   to divide reads into cell barcode groups.
2. cellSNP uses simple filtering for outputting SNPs, i.e., total UMIs or counts
   and minor alleles fractions. The idea here is to keep most information of 
   SNPs and the downstream statistical model can take the full use of it.
   
cellSNP has now a C version named cellsnp-lite_, which is basically more efficient 
with higher speed and less memory usage.

News
----
We recommend cellsnp-lite_ instead of cellSNP if you would like to use mode 2. For now, 
cellSNP mode 2 uses pileup() function of pysam, which would filter duplicates and orphan reads 
by default and may lead to unexpected coverage reduction in some cases. Compared to cellSNP, 
cellsnp-lite provides a more flexible reads filtering for mode 2 so you could tune the filtering
parameters on your demand.

We have turn off the PCR duplicate filtering by default (--maxFLAG), as it is not well flagged in 
CellRanger, hence may result in loss of a substantial fraction of SNPs. Please use v0.3.1 or setting 
--maxFLAG to large number. Credits to issue13_.

All release notes can be found in `doc/release.rst`_.

For computational efficiency, we initialised comments on this: `doc/speed.rst`_

.. _issue13: https://github.com/single-cell-genetics/cellSNP/issues/13
.. _doc/release.rst: https://github.com/single-cell-genetics/cellSNP/blob/master/doc/release.rst
.. _doc/speed.rst: https://github.com/single-cell-genetics/cellSNP/blob/master/doc/speed.rst

Installation
------------

cellSNP is available through `pypi`_. To install, type the following command 
line, and add ``-U`` for upgrading:

.. code-block:: bash

  pip install -U cellSNP

Alternatively, you can install from this GitHub repository for latest (often 
development) version by following command line

.. code-block:: bash

  pip install -U git+https://github.com/single-cell-genetics/cellSNP

In either case, if you don't have write permission for your current Python 
environment, we suggest creating a separate `conda environment`_ or add 
``--user`` for your current one.

.. _conda environment: https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html

Quick usage
-----------

Once installed, check all arguments by type ``cellSNP -h`` (see a snapshot_)
There are three modes of cellSNP:

* **Mode 1: pileup a list of SNPs for a single BAM/SAM file**

Use both `-R` and `-b`. 

Require: a single BAM/SAM file, e.g., from cellranger, a list of cell barcodes,
a VCF file for common SNPs. This mode is recommended comparing to mode 2, if a 
list of common SNP is known, e.g., human (see Candidate SNPs below)

.. code-block:: bash

  cellSNP -s $BAM -b $BARCODE -O $OUT_DIR -R $REGION_VCF -p 20 --minMAF 0.1 --minCOUNT 20
  
As shown in the above command line, we recommend filtering SNPs with <20UMIs  
or <10% minor alleles for downstream donor deconvolution, by adding 
``--minMAF 0.1 --minCOUNT 20``

Besides, special care needs to be taken when filtering PCR duplicates for scRNA-seq data by 
setting maxFLAG to a small value, for the upstream pipeline may mark each extra read sharing 
the same CB/UMI pair as PCR duplicate, which will result in most variant data being lost. 
Due to the reason above, cellSNP by default uses a large maxFLAG value to include PCR 
duplicates for scRNA-seq data when UMItag is turned on.

* **Mode 2: pileup whole chromosome(s) for a single BAM/SAM file**

Don't use `-R` but flexible on `-b`. 

This mode requires inputting a single bam file with either cell barcoded 
(add `-b`) or a bulk sample:

.. code-block:: bash

  # 10x sample with cell barcodes
  cellSNP -s $BAM -b $BARCODE -O $OUT_DIR -p 22 --minMAF 0.1 --minCOUNT 100

  # a bulk sample without cell barcodes and UMI tag
  cellSNP -s $bulkBAM -O $OUT_DIR -p 22 --minMAF 0.1 --minCOUNT 100 --UMItag None
  
Add `--chrom` if you only want to genotype specific chromosomes, e.g., `1,2`, 
or `chrMT`.

Recommend filtering SNPs with <100UMIs or <10% minor alleles for saving space
and speed up inference when pileup whole genome: ``--minMAF 0.1 --minCOUNT 100``

Note, this mode may output false positive SNPs, for example somatic variants or 
falses caussed by RNA editing. These false SNPs are probably not consistent in 
all cells within one individual, hence confounding the demultiplexing. 
Nevertheless, for species, e.g., zebrafish, without a good list of common SNPs, 
this strategy is still worth a good try, and it does not take much more time 
than mode 1.

Update: We recommend cellsnp-lite_ instead of cellSNP if you would like to use mode 2. For now, 
cellSNP mode 2 uses pileup() function of pysam, which would filter duplicates and orphan reads 
by default and may lead to unexpected coverage reduction in some cases. Compared to cellSNP, 
cellsnp-lite provides a more flexible reads filtering for mode 2 so you could tune the filtering
parameters on your demand.

* **Mode 3: pileup a list of SNPs for one or multiple BAM/SAM files**

Use `-R` but not `-b`.

Require: one or multiple BAM/SAM files (bulk or smart-seq), their according 
sample ids (optional), and a VCF file for a list of common SNPs. BAM/SAM files 
can be input in comma separated way (`-s`) or in a list file (`-S`). 

.. code-block:: bash

  cellSNP -s $BAM1,$BAM2,$BAM3 -I sample_id1,sample_id2,sample_id3 -o $OUT_FILE -R $REGION_VCF -p 20 --UMItag None

  cellSNP -S $BAM_list_file -I sample_list_file -o $OUT_FILE -R $REGION_VCF -p 20 --UMItag None

Set filtering thresholds according to the downstream analysis. Please add 
``--UMItag None`` if you bam file does not have UMIs, e.g., smart-seq and bulk 
RNA-seq.


List of candidate SNPs
----------------------

A quality list of candidate SNPs (ususally common SNPs) are important for mode 1
and mode 3. If a list of genotyped SNPs is available, it can be used to pile up.
Alternatively, for human, common SNPs in population that have been idenetified 
from consortiums can also be very good candidates, e.g., gnomAD_ and 
1000_Genome_Project_. For the latter, we have compiled a list of 7.4 million 
common variants (AF>5%) with this bash script_ and stored in this folder_.

In case you want to lift over SNP positions in vcf file from one genome build 
to another, see our `LiftOver_vcf`_ wrap function.


FAQ and releases
----------------
For troubleshooting, please have a look of `FAQ.rst`_, and we welcome reporting 
any issue_.

All releases are included in pypi_. Notes for each release are recorded in
`release.rst`_.


.. _vireo: https://github.com/huangyh09/vireo
.. _cellsnp-lite: https://github.com/single-cell-genetics/cellsnp-lite
.. _snapshot: https://github.com/huangyh09/cellSNP/blob/master/doc/manual.rst
.. _pysam: https://github.com/pysam-developers/pysam
.. _pypi: https://pypi.org/project/cellSNP/
.. _gnomAD: http://gnomad.broadinstitute.org
.. _1000_Genome_Project: http://www.internationalgenome.org
.. _script: https://github.com/huangyh09/cellSNP/blob/master/SNPlist_1Kgenome.sh
.. _folder: https://sourceforge.net/projects/cellsnp/files/SNPlist/
.. _LiftOver_vcf: https://github.com/huangyh09/cellSNP/tree/master/liftOver
.. _release.rst: https://github.com/huangyh09/cellSNP/blob/master/doc/release.rst
.. _FAQ.rst: https://github.com/huangyh09/cellSNP/blob/master/doc/FAQ.rst
.. _issue: https://github.com/huangyh09/cellSNP/issues
