# Protocol for processing 18Sv4 sequences in QIIME2

This document describes our procedure for processing 18Sv4 amplicon libraries using the V4F/V4RB primer set ([Balanzo et al. 2015](https://doi.org/10.3354/ame01740)). This primer set is very similar to those described by [Stoeck et al. 2010](https://doi.org/10.1111/j.1365-294X.2009.04480.x) except the last nucleotide on the reverse primer has been removed. This allows for amplification of a larger fraction of protists, particularly Haptophyta. Amplicon sequence variants are generated with DADA2 ([Callahan et al. 2016](https://doi.org/10.1038/nmeth.3869)). For annotation, we primarily use the [PR2 database](https://pr2-database.org/) ([Guillou et al. 2013](https://academic.oup.com/nar/article/41/D1/D597/1064851)).

QIIME2 visualizations can be viewed [here](https://view.qiime2.org/).

We normally but not exclusively generate these libraries from DNA and RNA extracted from Sterivex filters. See our automated extraction protocols here:
* [Sterivex DNA extraction](https://dx.doi.org/10.17504/protocols.io.bc2hiyb6)
* [Sterivex RNA extraction](https://dx.doi.org/10.17504/protocols.io.bd9ti96n)

## Requirements
* [QIIME2 Version 2019.10](https://docs.qiime2.org/2019.10/) or higher
* [PR2 database version 4.13.0](https://pr2-database.org/) or higher

## Important Notes
* This is a generic protocol. We often append a unique project ID to the beginning of all filenames throughout.
* Some of the specified options may not be appropriate and need to be adjusted according to your data and configuration. Consult the [QIIME2 documentation](https://docs.qiime2.org/2019.10/) and other software documentation where appropriate to fully understand what each command and option is doing. We have attempted to point out specific examples where parameters need to be adjusted but these should not be viewed as a comprehensive list.
* Refer to the QIIME2 documentation [here](https://docs.qiime2.org/2019.10/citation/) on how to properly cite QIIME2 and plugins used.

## Start

Activate your conda environment for QIIME2 and cd to your working directory

```
source activate qiime2-2019.10
cd $working_directory
```

## Import files

This assumes your data are provided as demultiplexed paired-end fastq files with PHRED 33 encoded quality scores. If your data are in a different format, see the [QIIME2 documentation on importing data](https://docs.qiime2.org/2019.10/tutorials/importing/).

First, generate a fastq manifest file which maps sample IDs to the full path of your fastq files (compressed as fastq.gz is also fine). The manifest is a tab-delimited file (.tsv) with the following headers:

```
sample-id forward-absolute-filepath reverse-absolute-filepath
```

Import the files and validate the output.

```
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path manifest.tsv \
  --output-path paired-end-demux.qza \
  --input-format PairedEndFastqManifestPhred33V2

qiime tools validate paired-end-demux.qza
```

A visualization of the imported sequences is often helpful.
```
qiime demux summarize \
  --i-data paired-end-demux.qza \
  --o-visualization paired-end-demux.qzv
```

## Trim reads

Remove the primers from reads with cutadapt.

* [QIIME2 cutadapt trim-paired documentation](https://docs.qiime2.org/2019.10/plugins/available/cutadapt/trim-paired/)
* [Cutadapt standalone documentation](https://cutadapt.readthedocs.io/en/stable/guide.html) ([Martin 2011](https://doi.org/10.14806/ej.17.1.200))

Parameter notes:
* This example shows trimming [linked adapters](https://cutadapt.readthedocs.io/en/stable/guide.html#linked-adapters-combined-5-and-3-adapter) as the amplicon is "framed" by both a 5' and 3' adapter. This is equivalent to usine the -a and -A options in cutadapt and shows the forward primer linked to the reverse complement of the reverse primer and vice versa. The --p-front-f and --p-front-r options may also be used.
* Number of CPU cores (--p-cores) can be increased up to the number of available cores.

```
qiime cutadapt trim-paired \
    --i-demultiplexed-sequences paired-end-demux.qza  \
    --p-cores 1 \
    --p-adapter-f ^CCAGCASCYGCGGTAATTCC...YRATCAAGAACGAAAGT \
    --p-adapter-r ^ACTTTCGTTCTTGATYR...GGAATTACCGCRGSTGCTGG \
    --p-error-rate 0.1 \
    --p-overlap 3 \
    --verbose \
    --o-trimmed-sequences paired-end-demux-trimmed.qza
```

Visualize trimmed sequences:

```
qiime demux summarize \
    --i-data paired-end-demux-trimmed.qza \
    --p-n 100000 \
    --o-visualization paired-end-demux-trimmed.qzv
```

## Denoise with DADA2

Generate and quantify amplicon sequence variants ASVs with DADA2

* [QIIME2 DADA2 denoise-paired docmentation](https://docs.qiime2.org/2019.10/plugins/available/dada2/denoise-paired/)
* DADA2 [website](https://benjjneb.github.io/dada2/index.html) and [FAQ](https://benjjneb.github.io/dada2/faq.html) ([Callahan et al. 2016](https://doi.org/10.1038/nmeth.3869))

Parameter notes:
* --p-n-threads is set to 0 which uses all available cores. Adjust accordingly.
* You may want to adjust the max-ee paramters (number of expected errors) depending on your data.
* --p-trunc-len-f and --p-trunc-len-r are based on typical read quality profiles we observe with MiSeq 2x300 sequencing. It is highly likely you should adjust these parameters for your own sequencing run. However, DADA2 requires a minimum of 20 nucleotides of overlap.
* --p-chimera-method consensus is the default and may perform better than pooled (See https://forum.qiime2.org/t/dada2-chimera-filtering-and-beyond/8685/3)
* If you have a large project that spans multiple sequence runs, run dada2 separately on each run. This is because different runs can have different error profiles (See https://benjjneb.github.io/dada2/bigdata.html). Since ASVs have single nucleotide level resolution, the data can later be merged (see instructions below). If merging data, ensure that your dada2 parameters are consistent. 

```
mkdir asvs

qiime dada2 denoise-paired \
    --i-demultiplexed-seqs paired-end-demux-trimmed.qza \
    --p-n-threads 0 \
    --p-trunc-q 2 \
    --p-trunc-len-f 262 \
    --p-trunc-len-r 223 \
    --p-max-ee-f 2 \
    --p-max-ee-r 2 \
    --p-n-reads-learn 1000000 \
    --p-chimera-method pooled \
    --o-table ./asvs/table-dada2.qza \
    --o-representative-sequences ./asvs/rep-seqs-dada2.qza \
    --o-denoising-stats ./asvs/stats-dada2.qza
```

Generate and examine the DADA2 stats. You should be retaining most (>50%) of your sequences. If you are losing a large number of sequences at a certain DADA2 step, you will need to troubleshoot and adjust your DADA2 parameters accordingly.

```
qiime metadata tabulate \
  --m-input-file ./asvs/stats-dada2.qza \
  --o-visualization ./asvs/stats-dada2.qzv
```

## Merge and summarize denoised data

If you have multiple sequencing runs, proceed with merging the table and sequences from separate dada2 runs. If not, proceed to the summarization and export steps.

Merge (add additional lines for --i-tables and --i-data as needed):

```
qiime feature-table merge \
  --i-tables ./asvs/run1_table-dada2.qza \
  --i-tables ./asvs/run2_table-dada2.qza \
  --o-merged-table merged_table-dada2.qza

qiime feature-table merge-seqs \
  --i-data ./asvs/run1_rep-seqs-dada2.qza \
  --i-data ./asvs/run2_rep-seqs-dada2.qza \
  --o-merged-data merged_rep-seqs.qza
```

Summarize and export:

```
qiime tools export \
  --input-path merged_table-dada2.qza \
  --output-path asv_table

biom convert -i asv_table/feature-table.biom -o asv_table/asv-table.tsv --to-tsv
```

```
qiime tools export \
  --input-path merged_rep-seqs.qza \
  --output-path asvs

qiime feature-table tabulate-seqs \
  --i-data merged_rep-seqs.qza \
  --o-visualization merged_rep-seqs.qzv
```

If you have metadata, you can include it here:

```
qiime feature-table summarize \
  --i-table merged_table-dada2.qza \
  --o-visualization merged_table-dada2.qzv \
  --m-sample-metadata-file metadata.tsv
```

## Taxonomic annotation

* The [Naive Bayes Classifier](https://docs.qiime2.org/2019.10/tutorials/feature-classifier/) is used here.
* We place the PR2 database files under a folder called 'db'

Import the sequence and taxonomy information. Select the region between the primers and train the classifier:

```
qiime tools import \
  --type 'FeatureData[Sequence]' \
  --input-path ./db/pr2/pr2_version_4.13.0_18S_mothur.fasta  \
  --output-path ./db/pr2/pr2_v4.13.0.qza

# import taxonomy
qiime tools import \
  --type 'FeatureData[Taxonomy]' \
  --input-format HeaderlessTSVTaxonomyFormat \
  --input-path ./db/pr2/pr2_version_4.13.0_18S_mothur.tax \
  --output-path ./db/pr2/pr2_v4.13.0_tax.qza

qiime feature-classifier extract-reads \
  --i-sequences ./db/pr2/pr2_v4.13.0.qza \
  --p-f-primer CCAGCASCYGCGGTAATTCC \
  --p-r-primer ACTTTCGTTCTTGATYR \
  --o-reads ./db/pr2/pr2_v4.13.0_v4_extracts.qza

# Train the classifier
qiime feature-classifier fit-classifier-naive-bayes \
  --i-reference-reads ./db/pr2/pr2_v4.13.0_v4_extracts.qza \
  --i-reference-taxonomy ./db/pr2/pr2_v4.13.0_tax.qza \
  --o-classifier ./db/pr2/pr2_v4.13.0_v4_classifier.qza
```

Classify ASVs. The option --p-n-jobs -1 uses the all CPUs. Adjust accordingly:

```
qiime feature-classifier classify-sklearn \
  --p-n-jobs -1 \
  --i-classifier ../db/pr2/pr2_v4.12.0_v4_classifier.qza \
  --i-reads merged_rep-seqs.qza \
  --o-classification merged_asv_tax_sklearn.qza

qiime tools export \
  --input-path merged_asv_tax_sklearn.qza \
  --output-path asv_tax_dir

mv asv_tax_dir/taxonomy.tsv asv_tax_dir/pr2_taxonomy.tsv
```

## Final output table

From here, you should be able to do additional analyses within QIIME2. If you would like to create an tab-delimited file with your ASV IDs, sample counts, and taxonomic classification, run the following in R:

```
# Read in asv table
asv_table <- read.delim("asv_table/asv-table.tsv", header=TRUE, row.names=NULL, stringsAsFactors=FALSE, na.strings = "n/a", skip = 1)
names(asv_table)[1] <- "Feature.ID"

pr2 <- read.delim('./asv_tax_dir/pr2_taxonomy.tsv', header=TRUE, row.names=NULL)
names(pr2)[2:3] <- paste("pr2", names(pr2)[2:3], sep="_")

output <- merge(asv_table, pr2, by="Feature.ID")

write.table(output, file="asv_count_tax.tsv", quote=FALSE, sep="\t", row.names=FALSE)
```
