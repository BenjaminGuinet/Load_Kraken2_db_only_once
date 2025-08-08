# Kraken2 Memory Optimization Guide

This guide explains how to use Kraken2 with memory mapping to load the database only once and then loop over the fastq files to classify them. 
This approach is particularly useful on a cluster where you can first copy the full database to a temporary node and then specify the location of the database to Kraken2.

Example of time spent of contigs against NT database : 
- For 466 of contig files (Tot size : 2.GB)
- Max time / file = 11 sec
- Min time /file = 215 sec
- Total time spent to classify : 6.4 hours


## Script Explanation

The following steps demonstrates how to achieve this:


## Step-by-Step Explanation

1. **Load Kraken2 Module**:
   ```bash
   module load kraken2/2.1.2
   ```

2. **Copy Database to Temporary Directory**:
   ```bash
   echo "Copying database to /tmp/..."
   cp -r path/to/Krakendb/k2_nt_20240530 /tmp/
   echo "Database copied to /tmp/."
   ```
   This step copies the Kraken2 database to the `/tmp/` directory, which is typically a temporary filesystem on a cluster node.
   On dardel all prebuilt databases are here ```/cfs/klemming/pdc/software/dardel/sw-uppmax/data/Kraken2_data/prebuilt/```

4. **Loop Over Fastq Files and run Kraken2 with Memory Mapping**:
   ```bash
   for input_file in /path/to/your/files/*.fasta.gz; do
   kraken2 --db /tmp/k2_nt_20240530 \
           --unclassified-out "${input_file%.fasta.gz}_unclass_nt.fq" \
           --report-minimizer-data \
           --threads 128 \
           --memory-mapping \
           --report "\${input_file%.fasta.gz}.kraken2_nt.report" \
           --output "\${input_file%.fasta.gz}.kraken2_nt.out" \
           ${input_file}
   ```
   The `--db` option specifies the location of the database in the temporary directory. The `--memory-mapping` option ensures that the database is loaded into memory only once.


## Example of full script counting the time used for each fastq file classified: 
```bash
module load kraken2/2.1.2

echo "Copying database to /tmp/..."
cp -r \$KRAKEN2_DB/k2_nt_20240530 /tmp/
echo "Database copied to /tmp/."

for input_file in *.fasta.gz; do
  # Start timing
  time_start=\$(date +%s)

  # Run Kraken2
  kraken2 --db /tmp/k2_nt_20240530 \
          --unclassified-out "\${input_file%.fasta.gz}_unclass_nt.fq" \
          --report-minimizer-data \
          --threads 128 \
          --memory-mapping \
          --report "\${input_file%.fasta.gz}.kraken2_nt.report" \
          --output "\${input_file%.fasta.gz}.kraken2_nt.out" \
          \${input_file}

  # Compress the output files
  pigz -p 128 "\${input_file%.fasta.gz}.kraken2_nt.out"
  pigz -p 128 "\${input_file%.fasta.gz}_unclass_nt.fq"

  # End timing
  time_end=$(date +%s)
  time_diff=$((time_end - time_start))

  # Output the time taken
  echo "Kraken2 successfully ran on unmapped reads..."
  echo "Time spent for running Kraken2 on unmapped reads of \${input_file%.fasta.gz}: \$time_diff seconds"
  echo ""
done
```

**Note :** For Dardel users, instead of the ```/tmp/``` folder you may have to cp to your scratch folder which should be found:
```/cfs/klemming/scratch/{first letter of username}/{username}``` in case you don't have enough space in tmp. 







