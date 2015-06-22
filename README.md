## Prepare data

Replace cumbersome filenames with symlinks.
Sample `YB1` becomes `s1`, and so on.

```bash
mkdir -p input/
for i in {1..8}
do
    ln -s /N/dc2/projects/brendelgroup/rice_genomes_BY/Illumina8/YB${i}/*1.fq.gz input/s${i}-1.fq.gz
    ln -s /N/dc2/projects/brendelgroup/rice_genomes_BY/Illumina8/YB${i}/*2.fq.gz input/s${i}-2.fq.gz
done
```

Interleave reads for deduplication and digital normalization.

```bash
for i in {1..8}
do
    paste <(gunzip -c input/s${i}-1.fq.gz | paste - - - -) \
          <(gunzip -c input/s${i}-2.fq.gz | paste - - - -) \
        | tr '\t' '\n' \
        | gzip -c - \
        > input/s${i}-int.fq.gz
done
wait
```

## Removal of PCR duplicates

Performed with [sequniq](https://github.com/standage/sequniq).

```bash
mkdir -p dedup/
git clone https://github.com/standage/sequniq.git
export PYTHONPATH=sequniq/  # Analogous to the `PERL5LIB` environmental variable
for i in {1..8}
do
    gunzip -c input/s${i}-int.fq.gz \
        | python sequniq/dedup.py \
        | gzip -c - \
        > dedup/s${i}-dedup-int.fq.gz &
done
wait  # Can remove this line if running in terminal interactively
```

## Digital normalization

Performed with [khmer](http://khmer.readthedocs.org/en/v1.4.1/).

```bash
mkdir -p diginorm/
for i in {1..8}
do
    normalize-by-median.py -k 20 -C 20 --paired \
                           -N 4 -x 6e9 \
                           -o diginorm/s${i}-diginorm-int.fq \
                           dedup/s${i}-dedup-int.fq.gz

    # Decouple interleaved reads for downstream steps
    paste - - - - - - - - < diginorm/s${i}-diginorm-int.fq \
        | tee >(cut -f 1-4 | tr '\t' '\n' | gzip -c - > diginorm/s${i}-diginorm-1.fq.gz) \
        | cut -f 5-8 | tr '\t' '\n' | gzip -c - > diginorm/s${i}-diginorm-2.fq.gz

    # Compress diginorm output for storage efficiency
    gzip diginorm/s${i}-diginorm-int.fq
done


```

## FastQC assessment

```bash
fastqc -t 16 diginorm/s?-diginorm-?.fq.gz
```

## Insert size estimation

Empirically confirm the insert size distribution for each library to make sure there were no library prep issues.
Used a 45kb region from chromosome 2 [34907172..34952509] which is pretty gene dense to map reads and measure observed insert size for each read pair.

```bash
module load java
module load picard
module load samtools
module load R

which bwa
which samtools
which java
which R

bwa index osat_chr2_34907172-34952509.fa
mkdir -p insert-size-est
cd insert-size-est

for i in {1..8}
do
  mkdir s${i}
  bwa mem -t 16 ../osat_chr2_34907172-34952509.fa ../diginorm/s${i}-diginorm-1.fq.gz ../diginorm/s${i}-diginorm-2.fq.gz \
      | samtools view -bS -F0x4 - \
      | samtools sort -o - bogus${i} \
      > s${i}/s${i}.bam
  java -jar /N/soft/mason/picard-tools-1.52/CollectInsertSizeMetrics.jar \
      HISTOGRAM_FILE=s${i}/inserts-hist-s${i}.pdf \
      INPUT=s${i}/s${i}.bam \
      OUTPUT=s${i}/inserts-hist-s${i}.out \
      > s${i}/metrics.log 2>&1
done

cd ..
```
