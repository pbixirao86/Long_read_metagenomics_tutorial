# Long_read_metagenomics_tutorial

Phase 1: Hardware Setup & Docker Installation
Before students arrive, they should follow these steps to turn their computers into a "bioinformatics workstation."

1. Install Docker Desktop
•	Download: Docker Desktop (Windows/Mac/Linux).
•	Install the app as an admin (right-click on installer and choose “run as administrator”)
•	Register on the docker website for an account
•	The app will likely open a terminal, where it will ask you to update you Linux subsystems – perform the update and restart PC

•	Windows Critical Step: Ensure WSL 2 is enabled during installation.

•	The "Safety Limit" (Crucial for 8GB RAM PCs):
•	In  “C:\Users\<YourUsername>\” you will find a file titled .wslconfig. Open and change the memory to 4GB, processors to 2 and the swap to 2GB 
•	It there is no .wslconfig in your home directory, you will need to create it: 
•	Open notepad 
•	Copy the following text into it: 

[wsl2]

memory=4GB     # Limits VM memory to 4GB

processors=2   # Limits VM to 2 logical processors

swap=2GB       # Sets swap space

•	Save the notepad as “.wslconfig” in your directory. Make sure you select “all files” under file type and keeping the inverted commas on the file name. 
•	Close the docker by typing on the terminal wsl –shutdown
•	To check if the restrictions on docker worked, type in the terminal: 

wsl  #this starts the docker
free -h #you should see the mem and swap rows showing the values you set

2. Prepare the Workshop Directory
Open a terminal (PowerShell on Windows, Terminal on Mac) and run:
Bash

# Create a folder structure to keep things tidy (in your username directory)
mkdir nanopore_workshop #making parent folder for the work we are going to perform

mkdir nanopore_workshop/data #make a directory within parent to store data

mkdir nanopore_workshop/results #make a directory for the result files

mkdir nanpore_workshop/databases #make directory for the kraken2 database

cd ~/nanopore_workshop/data

#Now that you are in your data folder, you can download the mock data we will be working with using the following command (on Windows):

Invoke-WebRequest -Uri "https://osf.io/download/buajf/" -OutFile "raw_reads.fastq.gz" 

curl.exe -L https://genome-idx.s3.amazonaws.com/kraken/k2_standard_08gb_20240904.tar.gz -o ./database/k2_std_8gb.tar.gz #the Kraken2 database (download prior to the tutorial, as it is fairly big – around 8 Gb) 

________________________________________
Phase 2: The "Low-Power" Pipeline
We will use lightweight tools that process data read-by-read rather than trying to build large genomic maps.
Step 1: Quality Control (NanoPlot)
Instead of processing millions of reads, we'll generate a summary report to see if the sequencing was successful.

Command line:

docker run --rm -v $(pwd):/data staphb/nanoplot:latest \
  NanoPlot --fastq /data/data/raw_reads.fastq.gz -o /data/results/qc_report

Step 2: Ultra-Fast Subsampling (Filtlong)
This is the "secret sauce" for low-power PCs. We will throw away the junk and keep only the best data, shrinking the file to a size the laptop can handle.

Command line:

docker run --rm -v $(pwd):/data staphb/filtlong:latest sh -c\
  “filtlong --min_length 1000 --target_bases 100000000 /data/data/raw_reads.fastq.gz > ./data/cleaned_reads.fastq”
# Note: This targets 100 Megabases, which is plenty for a training exercise.

Step 3: Fast Taxonomic Profiling (Kraken2)
To save RAM, we use a "Standard-8" database (capped at 8GB) and tell Kraken2 to map it to the disk rather than loading it all into memory.

Command line:

docker run --rm -v "${pwd}:/data" staphb/kraken2:latest sh -c "kraken2 --db /data/databases/standard_8gb --memory-mapping --threads 2 --report /data/results/taxonomy_report.txt /data/data/cleaned_reads.fastq > /data/results/kraken_output.txt"

Instructor Note: You should provide the standard_8gb database on a USB stick or a shared local drive. Downloading a metagenomics database during a workshop will kill the Wi-Fi.

Step 4: Visualizing Results (Krona)
Finally, convert the text output into an interactive "zoomable" pie chart that opens in any web browser.

Command line:

docker run --rm -v "${pwd}:/data" staphb/kraken2:latest sh -c "cut -f3,2 /data/results/kraken_output.txt > /data/results/krona_input.txt"

docker run --rm -v "${pwd}:/data" staphb/krona:latest ktImportText /data/results/krona_input.txt -o /data/results/interactive_chart.html 

________________________________________
Summary of the Workflow for Learners
Troubleshooting for Learners
•	"Permission Denied": On Linux, you might need to type sudo before every docker command.
•	"Out of Memory": If Step 3 crashes, it means the Docker Resource limit is set too low. Increase the memory in Docker Desktop to 5GB or 6GB.
•	Where are my files? Remember that $(pwd) means "Right here in my current folder." Everything they do in the container happens inside the /data folder, which is just a "mirror" of their workshop folder on their desktop.

Phase 3: Assembly & Mapping (The Low-RAM Way)


1. Lightweight Assembly (Raven)
•	Raven uses a very efficient "overlap" filter. It discards redundant data early in the process, meaning it doesn't have to keep as many millions of connections in the computer's RAM at once. 
•	While Raven does a bit of "polishing" (fixing errors), it is nowhere near as intensive as the multiple rounds of polishing MetaFlye performs. MetaFlye tries to achieve near-perfect accuracy for every species in a mix, which requires massive amounts of computation.
•	Raven is optimized for single-organism (or low-complexity) assemblies. MetaFlye has to run much more complex math to figure out which reads belong to "Species A" vs. "Species B" when their DNA sequences might be 95% identical.

Command line:

docker run --rm -v "${pwd}:/data" staphb/raven:latest sh -c "raven --threads 2 /data/data/cleaned_reads.fastq > /data/results/assembly.fasta" ________________________________________

Phase 4: Mapping Reads back to Assembly

Mapping is useful to see "Coverage"—how many of your original reads actually support the assembly you just made.
1. Generate the Alignment (BAM file)
We map the original cleaned reads back to our new assembly.fasta.

Command line:

docker run --rm -v "$(pwd):/data" staphb/minimap2:latest sh -c "minimap2 -ax map-ont /data/results/assembly.fasta /data/data/cleaned_reads.fastq > ./results/mapped_reads.sam"

2. Process and Sort (Samtools)
Computers can't read .sam files efficiently. We must convert them to binary (.bam) and sort them.

Command line:

docker run --rm -v $(pwd):/data staphb/samtools:latest \
  samtools sort /data/results/mapped_reads.sam -o /data/results/mapped_sorted.bam

3. Generate Statistics
Now, ask the computer to tell you how well the mapping went:

Command line:

docker run --rm -v "$(pwd):/data" staphb/samtools:latest sh -c "samtools flagstat /data/results/mapped_sorted.bam > /data/results/mapping_stats.txt"
________________________________________
Summary of the Complete "Low-Power" Pipeline
Why this works for your learners:
1.	Subsampling: By using only 50-100MB of data (Step 2 in the previous message), even Miniasm stays well under 2GB of RAM.
2.	No Consensus: Unlike Flye or Canu, Miniasm doesn't do "polishing" (which is the most RAM-heavy part). The resulting assembly will have a high error rate (~10%), but for a workshop exercise, it's perfect because students see a finished result in 5 minutes.
3.	Modular Docker: If a student's laptop is particularly slow, you can tell them to skip the assembly and simply provide them with a pre-computed assembly.fasta so they can practice the Mapping (Step 4) instead.

