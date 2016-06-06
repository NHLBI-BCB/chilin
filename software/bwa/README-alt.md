## Getting Started

Since version 0.7.11, BWA-MEM supports read mapping against a reference genome
with long alternative haplotypes present in separate ALT contigs. To use the
ALT-aware mode, users need to provide pairwise ALT-to-reference alignment in the
SAM format and rename the file to ""*idxbase*.alt". For GRCh38, this alignment
is available from the [BWA resource bundle for GRCh38][res].

#### Option 1: Mapping to the official GRCh38 with ALT contigs

Construct the index:
```sh
wget ftp://ftp.ncbi.nlm.nih.gov/genbank/genomes/Eukaryotes/vertebrates_mammals/Homo_sapiens/GRCh38/seqs_for_alignment_pipelines/GCA_000001405.15_GRCh38_full_analysis_set.fna.gz
gzip -d GCA_000001405.15_GRCh38_full_analysis_set.fna.gz
mv GCA_000001405.15_GRCh38_full_analysis_set.fna hs38a.fa
bwa index hs38a.fa
cp bwa-hs38-res/hs38d4.fa.alt hs38a.fa.alt
```

Perform mapping:
```sh
bwa mem hs38a.fa read1.fq read2.fq \
  | samblaster \
  | bwa-hs38-res/k8-linux bwa-postalt.js hs38a.fa.alt \
  | samtools view -bS - > aln.unsrt.bam
```
For short reads, the postprocessing script `bwa-postalt.js` runs at about the
same speed as BAM compression.

#### Option 2: Mapping to the collection of GRCh38, decoy and HLA genes

Construct the index:
```sh
cat hs38a.fa bwa-hs38-res/hs38d4-extra.fa > hs38d4.fa
bwa index hs38d4.fa
cp bwa-hs38-res/hs38d4.fa.alt .
```
Perform mapping:
```sh
bwa mem hs38d4.fa read1.fq read2.fq \
  | samblaster \
  | bwa-hs38-res/k8-linux bwa-postalt.js -p postinfo hs38d4.fa.alt \
  | samtools view -bS - > aln.unsrt.bam
```
This command line generates `postinfo.ctw` which loosely evaluates the presence
of an ALT contig with an empirical score at the last column.

**If you are not interested in the way BWA-MEM performs ALT mapping, you can
skip the rest of this documentation.**

## Background

GRCh38 ALT contigs are totaled 109Mb in length, spanning 60Mbp genomic regions.
However, sequences that are highly diverged from the primary assembly only
contribute a few million bp. Most subsequences of ALT contigs are highly similar
or identical to the primary assembly. If we align sequence reads to GRCh38+ALT
treating ALT equal to the primary assembly, we will get many reads with zero
mapping quality and lose variants on them. It is crucial to make the mapper
aware of ALTs.

BWA-MEM is designed to minimize the interference of ALT contigs such that on the
primary assembly, the ALT-aware alignment is highly similar to the alignment
without using ALT contigs in the index. This design choice makes it almost
always safe to map reads to GRCh38+ALT. Although we don't know yet how much
variations on ALT contigs contribute to phenotypes, we would not get the answer
without mapping large cohorts to these extra sequences. We hope our current
implementation encourages researchers to use ALT contigs soon and often.

## Methods

### Sequence alignment

As of now, ALT mapping is done in two separate steps: BWA-MEM mapping and
postprocessing.

#### Step 1: BWA-MEM mapping

At this step, BWA-MEM reads the ALT contig names from "*idxbase*.alt", ignoring
the ALT-to-ref alignment, and labels a potential hit as *ALT* or *non-ALT*,
depending on whether the hit lands on an ALT contig or not. BWA-MEM then reports
alignments and assigns mapQ following these two rules:

 * The original mapQ of a non-ALT hit is computed across non-ALT hits only.
   The reported mapQ of an ALT hit is computed across all hits.

 * An ALT hit is only reported if its score is strictly better than all
   overlapping non-ALT hits. A reported ALT hit is flagged with 0x800
   (supplementary) unless there are no non-ALT hits.

When option `-g FLOAT` is in use (which is the default), a third rule kicks in:

 * The mapQ of a non-ALT hit is reduced to zero if its score is less than FLOAT
   times the score of an overlapping ALT hit. In this case, the original mapQ is
   moved to the `om` tag.

If we don't care about ALT hits, we may actually skip postprocessing (step 2).
Nonetheless, postprocessing is recommended as it improves mapQ and gives more
information about ALT hits.

#### Step 2: Postprocessing

Postprocessing is done with a separate script `bwa-postalt.js`. It reads all
potential hits reported in the XA tag, lifts ALT hits to the chromosomal
positions using the ALT-to-ref alignment, groups them after lifting and then
reassigns mapQ based on the best scoring hit in each group with all the hits in
a group get the same mapQ. Knowing the ALT-to-ref alignment, this script can
greatly improve mapQ of ALT hits and occasionally improve mapQ of non-ALT hits.

The script also measures the presence of each ALT contig. For a group of
overlapping ALT contigs c_1, ..., c_m, the weight for c_k equals `\frac{\sum_j
P(c_k|r_j)}{\sum_j\max_i P(c_i|r_j)}`, where `P(c_k|r)=\frac{pow(4,s_k)}{\sum_i
pow(4,s_i)}` is the posterior of c_k given a read r mapped to it with a
Smith-Waterman score s_k. This weight is reported in `postinfo.ctw` in the
option 2 above.

### On the Completeness of GRCh38+ALT

While GRCh38 is much more complete than GRCh37, it is still missing some true
human sequences. To make sure every piece of sequence in the reference assembly
is correct, the [Genome Reference Consortium][grc] (GRC) require each ALT contig
to have enough support from multiple sources before considering to add it to the
reference assembly. This careful procedure has left out some sequences, one of
which is [this example][novel], a 10kb contig assembled from CHM1 short
reads and present also in NA12878. You can try [BLAT][blat] or [BLAST][blast] to
see where it maps.

For a more complete reference genome, we compiled a new set of decoy sequences
from GenBank clones and the de novo assembly of 254 public [SGDP][sgdp] samples.
The sequences are included in `hs38d4-extra.fa` from the [BWA resource bundle
for GRCh38][res].

In addition to decoy, we also put multiple alleles of HLA genes in
`hs38d4-extra.fa`. These genomic sequences were acquired from [IMGT/HLA][hladb],
version 3.18.0. Script `bwa-postalt.js` also helps to genotype HLA genes, though
not to high resolution for now.

## Problems and Future Development

There are some uncertainties about ALT mappings - we are not sure whether they
help biological discovery and don't know the best way to analyze them. Without
clear demand from downstream analyses, it is very difficult to design the
optimal mapping strategy. The current BWA-MEM method is just a start. If it
turns out to be useful in research, we will probably rewrite bwa-postalt.js in C
for performance; if not, we will try new designs. It is also possible that we
may make breakthrough on the representation of multiple genomes, in which case,
we can even get rid of ALT contigs once for all.


[res]: https://sourceforge.net/projects/bio-bwa/files/
[sb]: https://github.com/GregoryFaust/samblaster
[grc]: http://www.ncbi.nlm.nih.gov/projects/genome/assembly/grc/
[novel]: https://gist.github.com/lh3/9935148b71f04ba1a8cc
[blat]: https://genome.ucsc.edu/cgi-bin/hgBlat
[blast]: http://blast.st-va.ncbi.nlm.nih.gov/Blast.cgi?PROGRAM=blastn&PAGE_TYPE=BlastSearch&LINK_LOC=blasthome
[sgdp]: http://www.simonsfoundation.org/life-sciences/simons-genome-diversity-project/
[hladb]: http://www.ebi.ac.uk/ipd/imgt/hla/
