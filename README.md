# Workflow to sort Paired-ends fastq<br/>

Sometimes, we need to ENRICH and/or DEPLETE paired-ends Illumina data set for a particular target<br/>

Below are example command lines using BWA-mem + Samtools to do so<br/>

The Raw Illumina data files are ```raw_R1.fastq``` and ```raw_R2.fastq```<br/>
The file with scaffolds is ```query.fasta```<br/>

BWA-mem v0.7.15-r1140 and Samtools v1.6 are in the path on a Linux platform<br/>

**1) Index the file containing scaffolds**<br/>

```
bwa index query.fasta
```

**2) Run the actual read mapping and pipe results to samtools (adjust -t parameters to your number of cores)**<br/>

```
bwa mem -t 40 query.fasta raw_R1.fastq raw_R2.fastq | samtools sort > query.sorted.bam
```

**3) Index the sorted bam file (not mandatory but does not hurt if you plan to visualize the alignment in e.g. IGV)**<br/>

```
samtools index query.sorted.bam
```

# ENRICH target (skip unmapped reads)<br/>
**Enrich by skipping unmapped reads ```-F 4``` (thus keeping mapped reads)**<br/>

```
samtools view -b -F 4 query.sorted.bam > enriched.sorted.bam
```

(Use flags ```-f 2 -F 260``` if you want to only keep proper pairs and exclude unmapped reads + secondary alignments
or flags ```-f 2 -F 2308``` to exclude unmapped reads + secondary alignments + supplementary alignments. However, if you have circular molecules or structural variation, using ```-F 4 or -F 260``` is safer. My understanding is that for instance, reads mapping at the edge of a circular contig are considered supplementary alignment and would be excluded with the flag ```-F 2308```)<br/>

**Extract the reads to fastq file**<br/>

```
samtools fastq enriched.sorted.bam -1 enriched_R1.fastq -2 enriched_R2.fastq -0 /dev/null -n
```

# and/or DEPLETE target (keep unmapped reads)<br/>
**Deplete by keeping unmapped reads ```-f 4``` (thus skipping mapped reads)**<br/>

```
samtools view -b -f 4 query.sorted.bam > depleted.sorted.bam
```

(Unlike above, we cannot sort proper pairs since such information is gained via mapping and we are segregating unmapped reads...)

**Extract reads to fastq file**<br/>

```
samtools fastq depleted.sorted.bam -1 depleted_R1.fastq -2 depleted_R2.fastq -0 /dev/null -n
```

# COUNT reads in R1 fastq files to check<br/>
**Raw (initial) R1 fastq file**<br/>

```
cat raw_R1.fastq | echo -e "Rawfile_R1\t" $((`wc -l`/4)) > counts.txt
```

**Enriched R1 fastq file**<br/>

```
cat enriched_R1.fastq | echo -e "Enriched_R1\t" $((`wc -l`/4)) >> counts.txt
```

**Depleted R1 fastq file**<br/>

```
cat depleted_R1.fastq | echo -e "Depleted_R1\t" $((`wc -l`/4)) >> counts.txt
```

**Sum Enriched + Depleted R1 fastq files**<br/>

```
cat enriched_R1.fastq depleted_R1.fastq | echo -e "Enriched+Depleted\t" $((`wc -l`/4)) >> counts.txt
```

**Display counts on screen**<br/>

```
head counts.txt
```

The sum of enriched and depleted R1 reads counts should equal those of the raw R1 fastq file<br/>
(if you used flags ```-f 2 -F 260``` or ```-f 2 -F 2308``` to enrich, the sum may not be equal but less)<br/>

# Extra: Single-end fastq data<br/>

```
bwa index query.fasta
bwa mem -t 40 query.fasta raw_single_end.fastq | samtools sort > query.sorted.bam
samtools index query.sorted.bam
```
**Enrich** (use ```-F 4``` to exclude unmapped or ```-F 260``` or ```-F 2308``` for further sorting, see above)<br/> 
```
samtools view -b -F 4 query.sorted.bam > enriched.sorted.bam
samtools fastq enriched.sorted.bam -0 enriched_SE.fastq
```
Note that we use the flag -0

**Deplete**<br/>
```
samtools view -b -f 4 query.sorted.bam > depleted.sorted.bam
samtools fastq depleted.sorted.bam -0 depleted_SE.fastq
```
**Count** (as above replacing with the appropriate fastq file name)<br/>

# Extra: Single-end fasta data<br/>

Same as above except mapping is done from a single-end fasta file and output of samtools indicated with the ```fasta``` command

```
bwa mem -t 40 query.fasta raw_single_end.fasta | samtools sort > query.sorted.bam
```
```
samtools view -b -F 4 query.sorted.bam > enriched.sorted.bam
samtools fasta enriched.sorted.bam -0 enriched_SE.fasta
```
```
samtools view -b -f 4 query.sorted.bam > depleted.sorted.bam
samtools fasta depleted.sorted.bam -0 depleted_SE.fasta
```

# Cautionary note<br/>

For paired-end data sets, if the sorted fastq files produced are to be used for genome assembly, then it is preferable to adjust flags so that only proper read pairs are output to avoid issues with the assembler (i.e. maintaining synchronized paired-end files)<br/>

It is good practice to always check your ```.bam``` files prior and after sorting to visualize a summary of their content using the ```flagstat``` command of Samtools (i.e. numbers of total mapped reads, proper pairs, secondary alignments, etc)<br/> 

