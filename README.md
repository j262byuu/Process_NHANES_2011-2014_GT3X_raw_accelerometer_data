# My workflow on how to derive meaningful parameters (Step count, Sleep, Physical Activity, Circadian Rhythm etc...) from NHANES 2011-2014 RAW Accelerometer data

The NHANES 2011 and 2013 cycles have released raw accelerometer data:
- [2011 Cycle](https://ftp.cdc.gov/pub/pax_g/)
- [2013 Cycle](https://ftp.cdc.gov/pub/pax_h/)

NHANES is an excellent dataset because it is nationally representative (as long as survey weights are handled correctly 😄), free, and open access. Using these two cycles, you can generate some impactful papers ~~(Association between A and B among C / Prevalence and correlates of A among B / Trends in A among B)~~!

However, the raw accelerometer data must be processed to produce meaningful results. While the CDC has already processed the raw files using the [MIMS-unit algorithm](https://mhealthgroup.github.io/MIMSunit/), I prefer working from scratch. Below is a step-by-step guide to processing these files.

---

## Steps for Processing NHANES Raw Accelerometer Data

### Step 0: Download All Files
First, generate a download list in batches. For example, create a file called `NHANESBatch.txt` with the URLs of the files to be downloaded.

Sample NHANESBatch.txt
```bash
https://ftp.cdc.gov/pub/pax_h/80550.tar.bz2
https://ftp.cdc.gov/pub/pax_h/80552.tar.bz2
https://ftp.cdc.gov/pub/pax_h/80553.tar.bz2
https://ftp.cdc.gov/pub/pax_h/80555.tar.bz2
https://ftp.cdc.gov/pub/pax_h/80556.tar.bz2
https://ftp.cdc.gov/pub/pax_h/80559.tar.bz2
https://ftp.cdc.gov/pub/pax_h/80560.tar.bz2
https://ftp.cdc.gov/pub/pax_h/80561.tar.bz2
https://ftp.cdc.gov/pub/pax_h/80564.tar.bz2
https://ftp.cdc.gov/pub/pax_h/80565.tar.bz2
```

Use `aria2` for multithreaded downloading. Based on my experience, 8 threads work best—using more than 8 threads might result in a temporary ban from the CDC's FTP server. Below is an example command using an LSF environment:

```bash
bsub -G compute-snake -q general -R 'rusage[mem=4GB]' -a 'docker(hobbsau/aria2)' aria2c -i NHANESBatch1.txt -x 8 -s 8 -j 8
```

### Step 1: Decompress .bz2 Files and Delete the Originals

```bash
bsub -G compute-snake -q general -R 'rusage[mem=8GB]' -a 'docker(nestio/lbzip2)' lbzip2 -d *.bz2
```

After this step you will had a folder with a lot of tar files, file name is NHANES SEQN. 

### Step 2: Untar and Append All CSV Files
Here’s what needs to be done:

#### 1. Delete the `[SEQN]_Logs.csv` files.
These files are unnecessary as we'll do the QC (non-wear/imputation/sleep etc..) with other packages instead of [MIMS-unit algorithm](https://mhealthgroup.github.io/MIMSunit/).

#### 2. Append all CSV files
Append all relevant CSV files (e.g., `GT3XPLUS-AccelerationCalibrated-[FIRMWARE_VERSION].[SENSOR_ID].[YYYY]-[MM]-[DD]-[HH]-[MM]-[SS]-000-P0000.sensor.csv`) into a single, consolidated CSV file for each SEQN. Name the resulting file using the SEQN number for easy reference and organisation.

#### 3. Verify the append process
Generate a log file to confirm the merging process went smoothly, ensuring no data was missed or corrupted. As you know, appending a large number of CSV files can be risky, and doing so on a server is even more precarious.

For appending CSV files, there are several approaches. When working with high-frequency data (e.g., at the Hz level), I recommend using Linux. 
Why? Because Linux handles data processing efficiently and avoids issues with time or data formatting quirks.

```bash
bsub -G compute-snake -q general -R 'rusage[mem=8GB]' -a 'docker(eclipse/debian_jre)' bash mergeCSV.sh
```

What's inside mergeCSV.sh?

```bash
#!/bin/bash

# All csv location
baseDir="[Your tar folder]"
logFile="[Your log folder]"  # Path for the overall log file

# Create the log directory if it doesn't exist
mkdir -p "$(dirname "$logFile")"

# Clear the log file before appending new data
> "$logFile"

# Move to the base directory
cd "$baseDir" || exit

# Loop through each .tar file in the directory
for tarFile in *.tar; do
    # Extract NHANE SEQN
    SEQN=$(basename "$tarFile" .tar)
    
    # Create a folder named SEQN and extract the tar file into it
    mkdir "$SEQN"
    tar -xf "$tarFile" -C "$SEQN"
    
    # Save some space by removing the original tar file
    rm "$tarFile"
    
    # Move to the SEQN directory
    cd "$SEQN" || exit
    
    # Remove the log file
    rm "${SEQN}_Logs.csv"
    
    # Initialize total size
    totalSize=0
    
    # Calculate total size of all individual CSV files
    for csvFile in *.csv; do
        size=$(stat -c%s "$csvFile")
        totalSize=$((totalSize + size))
    done
    
    # Merge all CSV files into one, excluding header duplicates
    ls *.csv | xargs awk 'FNR==1 && NR!=1{next;}{print}' > "../${SEQN}.csv"
    
    # Calculate the size of the merged CSV file
    mergedSize=$(stat -c%s "../${SEQN}.csv")
    
    # Append the sizes to the overall log file
    echo "SEQN: $SEQN" >> "$logFile"
    echo "Total size of individual CSV files: $totalSize bytes" >> "$logFile"
    echo "Size of merged CSV file: $mergedSize bytes" >> "$logFile"
    
    # Compare the sizes and log the result
    sizeDiff=$((totalSize - mergedSize))
    # This is arbitrary, I think 5000 is fine
    if (( sizeDiff < 5000 && sizeDiff > -5000 )); then
        echo "Merge successful for SEQN $SEQN, sizes are approximately equal." >> "$logFile"
    else
        echo "Warning: Incomplete merge for SEQN $SEQN, size difference is $sizeDiff bytes." >> "$logFile"
    fi
    echo "-------------------------------------------" >> "$logFile"

    # Move the merged SEQN.csv to the main directory
    cd ..
    mv "${SEQN}.csv" "$baseDir"

    # Clean up the SEQN directory
    rm -rf "$SEQN"
done
```
After this step, you will have numerous large CSV files.

### Step 3: Process the data

 These packages are like a Swiss Army knife ~~(NOT DNANEXUS)~~ for accelerometer researchers. Each package has its strengths, and you can choose based on your specific research needs.

#### [GGIR](https://github.com/wadpac/GGIR)
GGIR is a comprehensive tool for sleep, physical activity, and circadian rhythm analysis. The latest version (3.1.7) even outputs summarized event data. If you’re using the Verisense step count algorithm ~~(Remember to use V2 parameter)~~, you’ll get cadence.

#### [mMARCH.AC](https://github.com/WeiGuoNIMH/mMARCH.AC)
This now works with NHANES data (after a few edits—originally, it relied way too heavily on the Java-based `read.xlsx`, which was overkill). See [this](https://github.com/j262byuu/mMARCH.AC)

#### [biobankAccelerometerAnalysis](https://github.com/OxWearables/biobankAccelerometerAnalysis)
For physical activity-related research, I find the results from this package particularly reliable. Most outputs are presented at the percentage level.

#### [stepcount](https://github.com/OxWearables/stepcount)
This is an excellent package for step count analysis. Both SSL and RF algorithms perform well. I’ve found the step count results here to be reliable ~~(Comparing to Verisense step count algorithm)~~.

#### [asleep](https://github.com/OxWearables/asleep)
This package offers interesting results, but it can be resource-intensive. I recommend allocating at least 16 GB of memory per session to ensure smooth processing.

#### [TLBC](https://github.com/cran/TLBC)
This package can be used to process NHANES data. While you can’t download the trained models, you can use [CAPTURE-24](https://github.com/OxWearables/capture24)

# If you have any questions or would like access to the processed data (it’s way too large to upload to GitHub), feel free to contact me on LinkedIn!


