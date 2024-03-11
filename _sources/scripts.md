# FYP Analysis Scripts

## Format RNA-seq file name

```bash
for i in `echo *.scafseq_200`; do
 
 filename=(sed -i 's/^([^_]+_){4}([^_]+)_([^_]+)$/$2/' $i);
 mv $i $filename.scafseq_200

done
```

## GeneMakrS-T

```bash
#SBATCH -n 20
#SBATCH -J gmst
#SBATCH -p cpunode
#SBATCH -e %J.error
#SBATCH --mail-type=ALL
#SBATCH --mail-user=xingyu.hu20@student.xjtlu.edu.cn

input_dir=/data/bio/huxy/1kite/00.data/

for i in `echo ${input_dir}*.scafseq_200`;do
    j=$(basename $i .scafseq_200)

    ~/geneMarkS-T/gmst.pl \
        --output $j \
        --format GFF \
        --fnn \
        --faa \
        --clean 1 \
        $i
        
done
```

## HMM search

### hmmsearch

```bash
#SBATCH -n 20
#SBATCH -J hmmsearch
#SBATCH -p cpunode
#SBATCH -e %J.error
#SBATCH --mail-type=ALL
#SBATCH --mail-user=xingyu.hu20@student.xjtlu.edu.cn

input_dir=/data/bio/huxy/1kite/01.gmst

for i in `ls ${input_dir}/*.faa`;do
    sample_name=$(basename $i .faa)
    hmmsearch \
        --cpu 20 \
        -o $sample_name.hmmout \
        --tblout $sample_name.hmmtblout \
        --notextw \
        --noali \
        --max \
        -E 1e-5 \
        /data/bio/huxy/1kite/02.kofamscan/profiles/K08769.hmm \
        $i
done
```

### HMM formatting

```bash

for i in `echo *.hmmtblout`;do

    filename=$i
    
    total_lines=$(wc -l < "$filename")

    # Subtract 10 from the total lines to get the range of lines to keep
    start_line=$((total_lines - 10))

    # Use head to extract the lines from the beginning up to start_line and overwrite the original file
    head -n "$start_line" "$filename" > temp_file && mv temp_file "$filename"

    sed -i '1,3d' $filename

done
```

## BLASTp

### Build local database

```bash
makeblastdb -in db.fasta -dbtype prot -out dbname
```

### BLASTp

```bash
input_dir=/data/bio/huxy/1kite/02.hmmoutput/K08769;
db_dir=/data/bio/huxy/1kite/database/UCP1_HS/UCP1;


for i in `ls $input_dir/*.hitSeq.faa`;do

    sample_name=$(basename $i .hitSeq.faa)
    
    blastp \
        -query $i \
        -out $sample_name.blastp.result \
        -db $db_dir \
        -outfmt 6 \
        -evalue 1e-10 \
        -num_threads 30 \
        -max_target_seqs 10

done
```

### Keep best hit

```bash
perl /data/bio/tangm/scripts/besthit.pl $i;
```

### Sort by length

```bash
sort -t $'\t' -k 4 -r -n $i.besthit > $i.tmp && mv $i.tmp $i.besthit
```

### Extract the highest two hits' ids
    
```bash
awk 'NR<=2 {print $1}' $i.besthit > $sample_id.length.top2.id
```

### Extract the aa sequence

```bash
perl /data/bio/tangm/scripts/extractFA.pl $sample_id.pident.top2.id $aa_dir/$sample_id.faa
```

### Delete extra information from fasta header

```bash
cut -d" " -f1 $sample_id.length.top2.id.fa > $sample_id.tmp && mv $sample_id.tmp $sample_id.length.top2.id.fa
```

### Add file name to seq id

```bash
perl /data/bio/tangm/scripts/addFilename2seqID.pl $sample_id.length.top2.id.fa

sed -i 's/\.length\.top2//g' $sample_id.length.top2.id.fa.IDmodified
```

## Tree constuction

### Combine all files together

```bash
cat *.IDmodified > UCP1.combined.fa
# Remember to add the human UCP1 as outgroup
```

### mafft alignment

```bash
mafft --maxiterate 1000 --localpair --thread 30 UCP1.combined.fa > UCP1.combined.aln.fa
```

### Build tree

```bash
FastTree UCP1.combined.aln.fa > UCP1.tree
```

<script src="https://utteranc.es/client.js"
        repo="hxyv/ucp1"
        issue-term="pathname"
        label="✨💬✨"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
