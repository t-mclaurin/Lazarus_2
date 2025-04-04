# Lazarus

**Lazarus** is a pipeline for the search and mapping of species presence in publicly
available metagenomic and environmental samples. It is robust to the varying levels of
reference material for target species and to the growing datasets available to search.

## Contents 
 - [Setup](#setup)
 - [Input](#input)
 - [Steps](#steps)
 - [Output](#output)

## Setup
You will need two directories: scripts and data.
```
mkdir lazarus/data
mkdir lazarus/scripts
```
The pipeline is designed for commands to be run from the /scripts directory. 
```
cd lazarus/scripts
```
## Input

Lazarus requires 4 pieces of input to work
1. A Boolean search string that describes the SRA entries relevant to your search
(eg "freshwater metagenome AND united kingdom")
2. A reference name for that search (eg "freshwater")
3. Target species by binomial name in two lists: A list of species known to have publicly available reference genomes
(taxids_with)
4. A list of species known **not** to have publicly available reference genomes

## Steps - Sourmash
#### Converting to Taxonomic ID from Binomial name
This script takes a list of species binomial names. (../data/species_list.txt). It expects the names line
deliniated, the same as copy and pasting from an excel file. 
```
nano ../data/species_list.txt
```
```
Bufo bufo
Anguilla anguilla
Phragmites australis
Schoenoplectus triqueter
Sympetrum striolatum
Margaritifera margaritifera
```
The point of this step is to check the names of the species are identifiied correctly 
and to work with ncbis ID system from then on. 
This script needs to be run twice, with the output files to be changed to:
```
taxids_with.txt
taxids_without.txt
```
The script is:
```
sbatch get_taxids.sh
```

#### Creating sourmash signatures from complete reference genomes. 
This requires the file "taxids_with.txt". It outputs sketched signatures to the signatures sub-directory of the data directory. 
It also requires the ncbi_datasets conda package activated. 
```
sbatch get_refgens.sh 
```
#### Creating sourmash signatures from the nucleotide database. 
This requires the file "taxids_without.txt". It outputs sketched signatures to the signatures sub-directory of the data directory. 
```
sbatch make_refgens.sh 
```

#### Creating a sourmash database. 
These signatures are grouped into a .zip file. In this exmaple the custome database will end up being called our_database.sbt.zip
```
sourmash index our_database ../data/signatures/*.sig
```
### Getting sample accessions
This script requires three user inputs, the boolean search function for the SRA database, a name for the search 
and wether or not the search has been done before. If it has, then the script will ammend the name of the old file with the date of when it was created, and make a new file with the name of the search that only contains accessions past that date (aka samples that have not been searched yet)
The two searches can later be easily combined. 
```
sbatch get_sample_accession.sh
```
To test: Once you have this list, you can retreive a random subsample with which to run the later stages as a test. (In this example the project was called freshwater.). This is useful for estimating runtime and memory usage before commiting to the entire search. 
```
shuf -n 20 ../data/freshwater.txt > ../data/short_freshwater.txt
```

### Running Search. - CPU and memory efficient option
This script finds, downloads, sketches, searches-in and then deletes each SRA entry in a list. 
It will output the the accession ID and results for each search to a csv file called output.csv
The accessions listed in the file that you names "file_name" (something like "file_name_short" if 
you used the shuf function above) needs to be changed to match
The soumrash database made before (ie our_database) needs to be named in the second to last line 
```
sbatch run_search_v2.sh
```
### Running Search - Faster and re-usable option
These two scripts 1) download and process all the SRA samples listed into a directory and then 2) runs gather on each and outputs the results.
They have been parallelised and run considerably quicker than the previous option. 
```
mkdir ../data/freshwater_sigs
```


First we ensure we have no repeats of SRA entreis already downloaded. 
```
comm -23 <(sort ../data/freshwater.txt) <(ls ../data/freshwater_sigs/*.sig | sed 's#.*/##; s/\.sig//g' | sort) > ../data/freshwater_new.txt
```
Then we run the download. The number of jobs needs to be set to equalthe number of accessions being downloaded, and the number of jobs at once should be kept relatively low so as not to overwhelm the SRA. 
```
wc -l ../data/freshwater_new.txt 
sbatch preget_samples_p.sh
```
Then we run the search. The number of jobs needs to be set to to equal the number of signatures in the target directory.
```
ls ../data/freshwater_sigs/ | wc -l
sbatch run_search_v5.sh
```
 The outputs are created seperately in order to prevent results from bieng misattributed due to a race condition, and so need to be combined at the end:
 ```
cat ../data/outputs/output_*.csv > output.csv
```


### Formatting the output
The output.csv file is in a human readable oriented format, and needs to be made into a true csv. This is done by running the code below in the command line. (script coming soon)
```
awk '
/^[A-Z0-9]+$/ { sample=$0; next }  # Capture Sample Accession
/^overlap/ { next }                 # Skip header row
/^---------/ { next }                # Skip separator row
/ found [0-9]+ matches total/ { next }  # Remove summary lines
/ the recovered matches hit/ { next }    # Remove summary lines
NF >= 5 {
    split($5, genome_accession, " ");  # Extract only the first word from column 6
    print sample "," $1 "," $2 "," $3 "," $4 "," genome_accession[1]
}
' output.csv > formatted_output.csv

```
## Steps - R

With the downloaded formatted_output we can now make some maps and summary statistics. 




