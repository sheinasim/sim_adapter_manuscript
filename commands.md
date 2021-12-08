# HiFiAdapterFilt 

## Usage:

```
bash hifiadapterfilt.sh [ -p sequence file Prefix ] [ -l minimum match Length to filter. Default=44 ] [ -m minimum Match percentage to filter. Default=97]  [ -t number of Threads for blastn. Default=8 ] [ -o Outdirectory prefix Default=. ]
```

# HiFiAdapterFilt functions deconstructed

## BLAST script to identify PacBio Blunt Adapter sequence in raw reads

```
blastn -db pacbio_vectors_db -query ${taxa}.fasta -num_threads 40 -task blastn -reward 1 -penalty -5 -gapopen 3 -gapextend 3 -dust no -soft_masking true -evalue .01 -searchsp 1750000000000 -outfmt 6 > ${taxa}.contaminant.blastout
```

## Print unique read IDs containing near exact matches

```
cat ${taxa}.contaminant.blastout | grep 'NGB00972' | awk -v OFS='\t' '{if ($3 >= 97 && $4 >= 44) print $1}' | sort -u > ${taxa}.blocklist
```

## Remove reads containing adapter sequence from .fastq file

```
cat ${taxa}.fastq | paste - - - - | grep -v -f ${taxa}.blocklist -F | tr "\t" "\n" > ${taxa}_filt.fastq
```

# Cutadapt-filt command 

```
for x in `ls ${taxa}.fastq | sed 's/.fastq//'`
do
    cutadapt -b "AAAAAAAAAAAAAAAAAATTAACGGAGGAGGAGGA;min_overlap=35" \
    -b "ATCTCTCTCTTTTCCTCCTCCTCCGTTGTTGTTGTTGAGAGAGAT;min_overlap=45" \
    --discard-trimmed -o ${x}.cutadaptfilt.fastq ${x}.fastq -j 40 --revcomp -e 0.1 2> ${x}.report.txt
done
```

# Assembly commands

## Default hifiasm command 

```
hifiasm -o ${taxa} -t 40 ${taxa}.fastq
```

## Default PB-IPA command 

```
ipa local --nthreads 12 --njobs 4 --run-dir ${taxa} -i ${taxa}.fastq
```

## Default HiCanu command

```
canu gridOptionsExecutive="--mem-per-cpu=48g" saveOverlaps=true 'stageDirectory=$TMPDIR' gridOptions="--partition=atlas -A ag100pest --time=2-00:00:00" -p $A -d $A genomesize=325M -pacbio-hifi "${taxa}".fastq
```
