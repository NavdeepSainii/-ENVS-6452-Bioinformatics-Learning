# Amplicon Sequencing Analysis - 16S

This document outlines the steps for analyzing microbial community composition, including quality control, trimming, filtering, taxonomic assignment, and visualization.

---
title: "Saini_Assignment_3"
author: "Navdeep"
date: "`r Sys.Date()`"
output: html_document
---


# Introduction
This document outlines the steps for analyzing microbial community composition, including quality control, trimming, filtering, taxonomic assignment, and visualization.





# Set the working directory
# This ensures R knows where to find and save files for this project.
# Replace "~/Desktop/Bio/Assignment3" with your specific directory path.

```{r}

path <- setwd("~/Desktop/Bio/Assignment3")

## The 'setwd()' function sets the working directory to the specified path.
# Use this command each time you restart R or if your computer sleeps and resumes.
# The working directory resets after closing R, so re-run this command if needed.
```


```{r}
# Always load the basic libraries at the beginning of your script
# These libraries are essential for data manipulation, visualization, and analysis.
library(ggplot2)
library(tidyverse)
library(scales)
library(RColorBrewer)
library(dada2); packageVersion("dada2")
```

```{r}
# List all files in the specified directory
# 'path' is the working directory set earlier using setwd()
list.files(path)
```


```{r}
# Step 1: Quality Control
# List and sort the forward and reverse FASTQ files
# This ensures the files are correctly paired for downstream processing.
fnFs <- sort(list.files(path, pattern="_R1_001.fastq", full.names = TRUE))
fnRs <- sort(list.files(path, pattern="_R2_001.fastq", full.names = TRUE))
```

```{r}
# Extract sample names to track each file pair by a unique identifier
# `basename()` removes the directory path, leaving only the filename.
# `strsplit()` splits the filename into parts based on the underscore (_) delimiter.
# `sapply()` selects the first part of the filename (the sample name).
sample.names <- sapply(strsplit(basename(fnFs), "_"), `[`, 1)
```

```{r}
# Visualize the quality profiles of forward and reverse reads
# This step generates plots of the quality scores for the first two forward and reverse FASTQ files.
plotQualityProfile(fnFs[1:2])
plotQualityProfile(fnRs[1:2])
```

```{r}
# Assign filenames for the filtered forward and reverse FASTQ files
# These files will be stored in a subdirectory named "filtered" within the working directory.

# Forward reads: Create file paths for filtered files
filtFs <- file.path(path, "filtered", paste0(sample.names, "_F_filt.fastq.gz"))

# Reverse reads: Create file paths for filtered files
filtRs <- file.path(path, "filtered", paste0(sample.names, "_R_filt.fastq.gz"))

# Assign sample names as the names for the filtered file paths
# This ensures the filtered files are correctly linked to their respective samples.
names(filtFs) <- sample.names
names(filtRs) <- sample.names
```


```{r}
# Perform quality filtering and trimming on the forward and reverse reads
# Outputs a summary table ('out') showing the number of reads retained after filtering

 
out <- filterAndTrim(fnFs, filtFs, fnRs, filtRs, 
                     truncLen=c(260,180),  # Truncate forward reads to 260 bases and reverse reads to 180 bases
                     maxN=0,               # Discard any reads with ambiguous bases (N's)
                     maxEE=c(2, 2),         # Allow a maximum of 2 expected errors per read for both forward and reverse reads
                     truncQ=2,             # Truncate reads at the first instance of a quality score of 2 or less
                     rm.phix=TRUE,         # Remove any reads that match the phiX genome (a common sequencing control)
                     compress=TRUE,        # Save output files in compressed format (gzipped)
                     multithread=TRUE)     # Use multiple CPU cores for faster processing

# View the output summary table
# 'out' shows the number of reads before and after filtering for each sample
View(out)
```

```{r}
# Learn the error rates for the filtered forward reads
errF <- learnErrors(filtFs, multithread=TRUE)

# Learn the error rates for the filtered reverse reads
errR <- learnErrors(filtRs, multithread=TRUE)

# Plot the estimated error rates for the forward reads
plotErrors(errF, nominalQ=TRUE)
```

```{r}
## # This step identifies real biological sequences (ASVs) and removes sequencing errors from both Forward and Reverse reads
dadaFs <- dada(filtFs, err=errF, multithread=TRUE) ## DENOISED FORWARD READS
dadaRs <- dada(filtRs, err=errR, multithread=TRUE) ## DENOISED REVERSE READS
```
```{r}
#dadaFs is the output from running the dada function on the forward reads (filtFs)
dadaFs[[1]] ##[[1]] indexing accesses the first element in the dadaFs list
```

```{r}
## The mergePairs function in DADA2 is used to merge forward and reverse reads for each sample.
mergers <- mergePairs(dadaFs, filtFs, dadaRs, filtRs, verbose=TRUE) # Verbose enables detailed output, providing more information about the merging process. 
# Inspect the merger data.frame from the first sample
head(mergers[[1]])
```
```{r}
seqtab <- makeSequenceTable(mergers) ## Creating a Sequence Table with makeSequenceTable
##seqtab is the output of sequance table
dim(seqtab) # to check the dimensions of the seqtab such as number of rows and columns
```
```{r}
write.csv(seqtab, "~/Desktop/Bio/Assignment3/seqtab.csv")
View(seqtab)
```

```{r}
# Extracts the unique sequences (amplicon sequence variants, or ASVs) from the sequence table (seqtab). ##nchar = number of characters (length of each sequence)
table(nchar(getSequences(seqtab)))
```

```{r}
##This removes chimeric sequences from the sequence table 
seqtab.nochim <- removeBimeraDenovo(seqtab, method="consensus", multithread=TRUE, verbose=TRUE)
dim(seqtab.nochim)
```

```{r}
##Calculates the proportion of non-chimeric sequence
##seqtab.nochim : represnts sequnecs left after removing chimera and seqtab represents the original number of sequences = proportion of sequences retained after chimera removal
sum(seqtab.nochim)/sum(seqtab)
```


```{r}
write.csv(seqtab.nochim, file = "~/Desktop/Bio/Assignment3/seqtab.nochim.csv")
```

```{r}
# To get the number of unique sequences for a sample
#getN is a function to get the number of unique sequences
getN <- function(x) sum(getUniques(x))
#track is a function that gives table that summarizes the squences from each sample at each stage of the Dada2 Pipeline
#cbind combines columns of data in a table,"sapply" is applying function getN on objects dadaFs and dadaRs and mergers (merged paired end reads)
track <- cbind(out, sapply(dadaFs, getN), sapply(dadaRs, getN), sapply(mergers, getN), rowSums(seqtab.nochim)) ##  Calculates the total count of non-chimeric sequences for each sample.
# If processing a single sample, remove the sapply calls: e.g. replace sapply(dadaFs, getN) with getN(dadaFs)
colnames(track) <- c("input", "filtered", "denoisedF", "denoisedR", "merged", "nonchim")
##Assigns the sample names to each row in track, making it clear which row corresponds to which sample.
rownames(track) <- sample.names
head(track)
```


```{r}
##using reference database from "dada2" website for taxonomy
taxa <- assignTaxonomy(seqtab.nochim, "silva_nr99_v138.1_train_set.fa", multithread =TRUE)
```

```{r}
##using reference database from "dada2" website for species
taxa <- addSpecies(taxa, "silva_species_assignment_v138.1.fa")
```


```{r}
# Load necessary libraries for microbial analysis and visualization
# These libraries are essential for processing, analyzing, and visualizing microbiome data


#### Prerequisites
#Ensure all libraries are installed before running the script. Use the following commands to install them if necessary:

##install.packages(c("ggplot2", "dplyr"))
##if (!requireNamespace("BiocManager", quietly = TRUE))
    ##install.packages("BiocManager")
##BiocManager::install(c("phyloseq", "Biostrings"))

library(phyloseq); packageVersion("phyloseq")
library(Biostrings); packageVersion("Biostrings")
library(ggplot2); packageVersion("ggplot2")
library(dplyr)
```

```{r}
#Saving as you go, just in case if in future we need to start from these objects
##Save output as taxa as a csv file to my computer

write.csv (taxa,file = "~/Desktop/Bio/Assignment3/taxa.csv")

write.csv (seqtab.nochim,file = "~/Desktop/Bio/Assignment3/topseq.nochim.csv")

write.csv (track,file = "~/Desktop/Bio/Assignment3/track.csv")
```


```{r}

##### what file I want to work with 
taxa <- read.csv ("~/Desktop/Bio/Assignment3/taxa.csv")
seqtab.nochim <- read.csv ("~/Desktop/Bio/Assignment3/topseq.nochim.csv")
```

```{r}
###This is to transpose data in case if I have to do that 
flipped_seqtab.nochim <- as.data.frame(t(seqtab.nochim))
View(flipped_seqtab.nochim)
```


```{r}
#####To take the sample id from a 1st row to the first(header) as a name so because we don't wnt the numbers to be treated as a result of the sample however, instead of numbers of the sample ID make it as a name of the sample id 
####This command copied the first row of the data frame


names(flipped_seqtab.nochim) <- flipped_seqtab.nochim[1,]


##### copying the first row means it is 2 times in the data frame now so the one has to be deleted
##Deleting the first now


flipped_seqtab.nochim <- flipped_seqtab.nochim[-1,]
write.csv(flipped_seqtab.nochim, file = "~/Desktop/Bio/Assignment3/flipped_seqtab.nochim.csv")


##Now to merge two files, in this case merging seqtab with taxa is 
OTUabund <- cbind(flipped_seqtab.nochim, taxa)
write.csv(OTUabund, file = "~/Desktop/Bio/Assignment3/OTUabund.csv")

rownames(flipped_seqtab.nochim) <- paste0("ASV", 1:nrow(flipped_seqtab.nochim))
```

```{r}
##Converts the seqtab.nochim object, which likely is a matrix or another type of R object, into a data frame.
seqtab.nochim <- data.frame(seqtab.nochim)
## Extracts the row names from the seqtab.nochim data frame and stores them in samples.out.
samples.out <- rownames(seqtab.nochim)

##Creates a new data frame (samdf) that contains just one column, which is samples.out (the sample names).
samdf <- data.frame(samples.out)

#Sets the row names of the samdf data frame to be the same as the values in samples.out.
rownames(samdf) <- samples.out

##Converts the samdf data frame into a matrix.
samdf <- as.matrix(samdf)
```

```{r}
##Load OTU abundance data (OTUmat.csv) and taxonomy data (taxamat.csv) from CSV files into R.
##The data is initially stored as data frames because by default when R reads csv file it reads data as data frame which means that they could have different data types for each column (e.g., character, numeric). To use them in phyloseq, they need to be converted to matrices with numerical data.

uparse_otus <- read.csv("~/Desktop/Bio/Assignment3/OTUmat.csv")

uparse_taxa <- read.csv("~/Desktop/Bio/Assignment3/taxamat.csv")
 

##Purpose: Convert the OTU table (uparse_otus) and taxonomy table (uparse_taxa) from data frames into matrices.
#Phyloseq package requires the abundance and taxonomy data in a matrix form.
###Note: Converting to a matrix ensures that all data in the table is of a single type, typically numeric for abundance data, which is crucial for calculations.

# Convert the OTU table and taxonomy table from data frames into matrices
otumat <- as.matrix(uparse_otus)
taxamat <- as.matrix(uparse_taxa)



# Assign unique row names to each OTU, such as "OTU1", "OTU2", etc.
rownames(otumat) <- paste0("OTU", 1:nrow(otumat))
rownames(taxamat) <- paste0("OTU", 1:nrow(taxamat))
 

##This check ensures that both the abundance table (OTU counts) and the taxonomy table have the same number of rows. If they don’t match, phyloseq can’t connect abundance counts with their taxonomy, leading to an error.

if (nrow(otumat) != nrow(taxamat)) {
  stop("The number of OTUs in the abundance and taxonomy tables are not the same!")
}


##To check the structure of the object
str(otumat)  
str(taxamat) 


class(otumat)
 
class(taxamat)

class(uparse_otus)
class(uparse_taxa)
```


```{r}
# Create phyloseq components
OTU <- otu_table(otumat, taxa_are_rows = TRUE) 
TAX <- tax_table(taxamat)


# Create a phyloseq object
ps <- phyloseq(OTU, TAX)
```

```{r}
## Viewing this object gives an output of how many rows otu_table and taxa table has 
ps
```

```{r}
sample_names(ps)
```


```{r}
# Get sample names from the ps object
samplenames <- sample_names(ps)
print(samplenames)
```




```{r graphs}

##Active graphs interface in R studio yo make it working in chunks
### Plotting of Taxa by Phylum###
pp <- plot_bar(ps, fill = "Phylum") +
  ggtitle("Relative Abundance by Phylum") +
  theme_minimal() +
  xlab("Samples") +
  ylab("Abundance")
```

```{r}
#To view the plot
plot(pp)
```

```{r}
## Plot had NA in Phyllum so I plan to filter the Phylum column by removing NA's

ps_filtered <- subset_taxa(ps, !is.na(Phylum))
```

```{r}

### I personally like confirming if I remove the NA's successfully, so this is how you can do that## returning NA's shows no NA's

any(is.na(tax_table(ps_filtered)[, "Phylum"]))
```

```{r}
#### Plotting again
pp_filtered <- plot_bar(ps_filtered, fill = "Phylum") +
  ggtitle("Relative Abundance by Phylum") +
  theme_minimal() +
  xlab("Samples") +
  ylab("Abundance")
```

```{r}
# Plot the filtered data to visualize and validate filtering results
# 'pp_filtered' should be an object containing filtered data (e.g., from quality filtering or normalization steps).
plot(pp_filtered)
```

```{r}
# Create a stacked bar plot of microbial taxa at the Phylum level
# 'ps_filtered' is the filtered phyloseq object containing sample data and taxonomy, with NO NA's
pstacked <- plot_bar(ps_filtered, fill = "Phylum") +
  ggtitle("Stacked Bar Plot by Phylum") +
  theme_minimal() +
  xlab("Samples") +
  ylab("Abundance")

#Displays the stacked plot
plot(pstacked)
```

```{r}
#Aggregates data at the **Phylum** level using `tax_glom()`
# This command combines the counts for all ASVs/OTUs that belong to the same Phylum.
# It simplifies the dataset by grouping lower-level features into broader taxonomic categories (Phyla).
# The result is a new phyloseq object with data aggregated at the Phylum level.
ps_phylum <- tax_glom(ps_filtered, "Phylum")
```

```{r}
# Create a bar plot to visualize the abundance of taxa at the Phylum level
# 'ps_phylum' is the phyloseq object aggregated at the Phylum level
ps_phylum_glommed <- plot_bar(ps_phylum, fill = "Phylum")
```

```{r}
# Display the plot
plot(ps_phylum_glommed)
```

```{r}
# Transform counts to relative abundances
# `transform_sample_counts` applies a function to transform the counts in each sample.
# In this case, we calculate relative abundance by dividing each ASV count by the total count for the sample.

ps_phylum_relabun <- transform_sample_counts(ps_phylum, function(ASV) ASV/sum(ASV))
```

```{r}

# Each row in the resulting data frame corresponds to a single ASV, its sample, and associated taxonomic information.

taxa_abundance_table_phylum <- psmelt(ps_phylum_relabun)

# Convert the "Phylum" column into a factor
# This ensures that Phylum is treated as categorical data in visualizations and analyses.

taxa_abundance_table_phylum$Phylum <- factor(taxa_abundance_table_phylum$Phylum)
```

```{r}

# Create a bar plot of relative abundance by Phylum
# 'ps_phylum_relabun' contains the relative abundance data at the Phylum level

ps_RA <- plot_bar(ps_phylum_relabun, fill = "Phylum") +
  ggtitle("Relative Abundance of Taxa by Phylum") +
  theme_minimal() +
  xlab("Samples") +
  ylab("Relative Abundance")
```

```{r}
# Display the plot
ps_RA
```

```{r}
# Plot the absolute abundance of microbial taxa at the Order level
# 'ps' is the phyloseq object containing the abundance data
ps_order <- plot_bar(ps, fill = "Order") +
  ggtitle("Absolute Abundance by order") +
  theme_minimal() +
  xlab("Samples") +
  ylab("Abundance")
```

```{r}
# Render the plot in the RStudio Viewer or an external graphics device
ps_order
```

```{r}
# //ly, to Phylum - Plot had NA in Order so I plan to filter the Order column by removing NA's

ps_order_filtered <- subset_taxa(ps, !is.na(Order))
```


```{r}
#### Plotting again
pp_order_filtered <- plot_bar(ps_order_filtered, fill = "Order") +
  ggtitle("Absolute Abundance by Order") +
  theme_minimal() +
  xlab("Samples") +
  ylab("Abundance")

#Displays the filtered object
pp_order_filtered
```


```{r}
# Create a stacked bar plot of microbial taxa at the Order level
pp_order_stacked <- plot_bar(ps_order_filtered, fill = "Order") +
  ggtitle("Stacked Bar Plot by Order") +
  theme_minimal() +
  xlab("Samples") +
  ylab("Abundance")

#Displays the stacked 
pp_order_stacked
```

```{r}
# Aggregate data at the Order level
# This command combines the counts for all ASVs/OTUs that belong to the same Order.

ps_order <- tax_glom(ps_order_filtered, "Order")
```

```{r}
ps_order_glommed <- plot_bar(ps_order, fill = "Order")

plot(ps_order_glommed)
```


```{r}
##transform_sample_counts applies a function to transform teh counts for each sample
ps_order_relabun <- transform_sample_counts(ps_order, function(ASV) ASV/sum(ASV))
```

```{r}
#### Flattens the phyloseq object into a long-format tidy data frame, where each row corresponds to a single ASV, sample, and its associated taxonomic information.
taxa_abundance_table_order <- psmelt(ps_order_relabun)
###ensure that categorical data, like taxonomic ranks, are treated appropriately in visualizations.
taxa_abundance_table_order$Order <- factor(taxa_abundance_table_order$Order)
```

```{r}
## Here comes Plotting :)
ps_RA_order <- plot_bar(ps_order_relabun, fill = "Order") +
  ggtitle("Relative Abundance of Taxa by Order") +
  theme_minimal() +
  xlab("Samples") +
  ylab("Relative Abundance")

#Display
ps_RA_order
```

```{r}
# Identify shared ASVs between P1 and Wneg
shared_asvs <- taxa_abundance_table_phylum %>%
  filter(Sample %in% c("P1", "Wneg")) %>%  # Keep rows for P1 and Wneg only
  group_by(OTU) %>%                        # Group by ASV (OTU column)
  filter(n_distinct(Sample) == 2)          # Retain ASVs present in both samples

print(shared_asvs)
```

```{r}
## I wanted to try point graph instead of bar plot to plot relative abundance of taxa by sample

# Flatten phyloseq object to a tidy data frame
taxa_abundance_table <- psmelt(ps_phylum_relabun)  # Replace with your phyloseq object

# I am taking ataxa that are more than 0 in abundance,so Filter for a specific taxonomic rank (optional)
filtered_data <- taxa_abundance_table %>%
  filter(Abundance > 0)  # Keep only ASVs with non-zero abundance
```

```{r}
# To view entire color set of color_brewer library
display.brewer.all()
```
```{r}
# Create a geom_point plot
RA_sample_geom_point <- ggplot(taxa_abundance_table, aes(x = Sample, y = Phylum, size = Abundance, color = Phylum)) +
  geom_point(alpha = 0.8) +
  theme_minimal() +
  xlab("Samples") +
  ylab("Phylum") +
  ggtitle("Relative Abundance of ASVs by Sample") +
  scale_size(range = c(2, 10)) +
  scale_color_brewer(palette = "Paired")
  theme(axis.text.x = element_text(angle = 45, hjust = 1))

  ##Displays the plot
RA_sample_geom_point
```

