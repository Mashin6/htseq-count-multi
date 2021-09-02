# htseq-count-multi
htseq-count-multi is a modified version of htseq-count script that is part of [HTSeq python package](https://github.com/htseq/htseq) that allows multiple countings of the same .sam file in one pass.

Original htseq-count script allows counting of multiple .sam files at the same time, but does not allow performing multiple different types of countings on a single .sam file. This modified version allows arbitrary number of comparisons on single .sam input file.

## How to run
Parameters for different counting types are provided as comma separated list.
e.g. performing 3 counting jobs: 1) genes union gene_id 2) exon union gene_id 3) exon intersect strict gene_id
``` bash
python htseq-count-multi.py \
                        -f sam \
                        --samout output.sam \
                        -t gene,exon,exon \
                        -i gene_id,gene_id,gene_id \
                        -m union,union,intersection-strict \
                        -c GF_counts.txt,EF_counts.txt,XF_counts.txt \
                        input.sam \
                        annotation.gtf
```

Counts are returned in separate text files.<br>
Reads are returned as a single .sam file with each read containing new flags: GF:Z, [EF:Z, XF:Z, XG:Z, XH:Z, ...]


## Parallel run
htseq-count runs as single thread job. To run it in multithread mode one can use GNU parallel
``` bash
    samtools view -h -@ 20 input.bam \
        | parallel \
            -L 2 \
            -j 20 \
            --roundrobin \
            --header '(@.*\n)*' \
            --pipe python htseq-count-multi.py \
                        -f sam \
                        --samout output.{#}_temp.sam \
                        -t gene,exon,exon \
                        -i gene_id,gene_id,gene_id \
                        -m union,union,intersection-strict \
                        -c GF_counts.{#}_temp.txt,EF_counts.{#}_temp.txt,XF_counts.{#}_temp.txt \
                        input.sam \
                        annotation.gtf   
```


Files from individual jobs can be then merged together
```bash
# .sam files
samtools view -H input.bam \
        | cat - output.*_temp.sam \
        | $SAMTOOLS sort \
            -@ 20 \
            -n \
            -o output.bam
            
 # counts files
parallel -j 3 "awk -v 'OFS=\t' '
                                  {
                                        gene[\$1] += \$2
                                  }
                                  END {
                                         for (name in gene) {
                                            print name, gene[name] 
                                       }
                                  }' {1}_counts.*_temp.txt  \
                     | LC_COLLATE=C sort -k1,1 > {1}_counts.txt" ::: GF EF XF 
            
```



