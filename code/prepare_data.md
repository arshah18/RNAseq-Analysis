# Computing Different Normalization Schemes in R

```{r}
# The original data can found from recount2 database (https://jhubiostatistics.shinyapps.io/recount/) using SRA project code SRP029880.
# colorectal cancer
counts_file <- system.file('extdata/rna-seq/SRP029880.raw_counts.tsv',package = 'compGenomRData')
coldata_file <- system.file('extdata/rna-seq/SRP029880.colData.tsv',package = 'compGenomRData')
counts <- as.matrix(read.table(counts_file, header = T, sep = '\t'))
```
# Computing Counts Per Million (CPM)
```{r}
cpm <- apply(subset(counts, select = c(-width)), 2 ,function(x) x/sum(as.numeric(x)) * 10^6)
colSums(cpm)
```
# Computing Reads Per Kilobase Million (RPKM)
```{r}
# Create a vector of gene lengths
geneLengths <- as.vector(subset(counts, select = c(width)))

# Compute the rpkm
rpkm <- apply(X = subset(counts, select = c(-width)), MARGIN = 2, FUN = function(x) {
                  10^9 * x / geneLengths / sum(as.numeric(x))})
colSums(rpkm)
```
# Computing Transcripts Per Million (TPM)
```{r}
# Find gene length normalised values
rpk <- apply( subset(counts, select = c(-width)), 2 , function(x) x/(geneLengths/1000))
# normalised by sample size using rpk values
tpm <- apply(rpk, 2 , function(x) x / sum(as.numeric(x)) * 10^6)
colSums(tpm)
```
# Clustering
```{r}
# Let’s select the top 100 most variable genes among the samples
# compute the variance of each gene across samples
V <- apply(tpm, 1, var)
# sort the results by variance in decreasing order
# and select the top 100 genes
selectedGenes <- names(V[order(V, decreasing = T)][1:100])
# we can quickly produce a heatmap where samples and genes are clustered
pm, 1, var)
# Sort the results by variance in decreasing order
#and select the top 100 genes

selectedGenes <- names(V[order(V, decreasing = T)][1:100])

# we can quickly produce a heatmap where samples and genes are clustered
library(pheatmap)
pheatmap(tpm[selectedGenes,], scale = 'row', show_rownames = FALSE)
# We can also overlay some annotation tracks to observe the clusters
colData <- read.table(coldata_file, header = T, sep = '\t',
stringsAsFactors = TRUE)
pheatmap(tpm[selectedGenes,], scale = 'row',
show_rownames = FALSE,
annotation_col = colData)
```
# PCA Plots
```{r}
# Let’s make a PCA plot to see the clustering of replicates as a scatter plot in two dimensions
library(stats)
library(ggplot2)
library(ggfortify)
M <- t(tpm[selectedGenes,])
# transform the counts to log2 scale
M <- log2(M + 1)
# compute PCA
pcaResults <- prcomp(M)
# plot PCA results making use of ggplot2's autoplot function
# ggfortify is needed to let ggplot2 know about PCA data structure.
autoplot(pcaResults, data = colData, colour = 'group')
summary(pcaResults)
```
# Corellation Plots
```{r}
correlationMatrix <- cor(tpm)
library(corrplot)
corrplot(correlationMatrix, order = 'hclust' ,
              addrect = 2, addCoef.col = 'white',
              number.cex = 0.7)
# We could also plot this correlation matrix as a heatmap. Heatmap instead of a corrplot helps to see the differences between samples more easily. The annotation_col argument helps to display sample annotations and the cutree_cols argument is set to 2 to split the clusters into two groups based on the hierarchical clustering results.
# split the clusters into two based on the clustering similarity
pheatmap(correlationMatrix, annotation_col = colData, cutree_cols = 2)
## Figure will show the pairwise correlation of samples displayed as a heatmap
```

# Differential Expression Analysis
```{r}
Differential expression analysis allows us to test tens of thousands of hypotheses (one test for each gene) against null hypotheses that the activity of the gene stays the same in two different conditions.
## How DESeq2 workflow calculates differential expression:
1. The read counts are normalized by computing size factors, which address the differences not only in the library sizes, but also the library compositions.
2. For each gene, a disperssion estimate is calculated. The disperssion value computed by DESeq2 is equal to the squared coefficient of variation (variation divided by the mean)
3. A line is fit accorss the disperssion estimates of all the genes computed in step 2 versus the mean normaized counts of the genes.
4.Disperssion values of each gene are shrunk towards the fitted line in step 3.
5. A generalised Linear Model is fiited which considers additional confounding variables related to the experimental design such as sequencing batches, treatment, temperature, patients age, sequencing technology, etc., and uses the negative binomial distribution for fitting count data.
6. For a givem contrast (e.g. treatment type: drug-A versus untreated), a test for differential expression is carried out against the null hypothesis that log fold change of normalised counts of a gene in the given pair of groups is exactly zero.
7. It adjusts p-values for mutiple-testing.
In order to carry DEA using DESeq2, three kinds of inputs are necessary:
1. Read count table: raw read counts as integers that are not processed in any form by normalization technique. The rows represent feautures (e.g. genes, transcripts, genomic intervals) and columns represent samples.
2. coldata table: defines the experimental design.
3. design formula: needed to describe variable of interest in the analysis (e.g treatment status) along with (optionally) other covariates (e.g. batch, temperature, sequencing technology).
s (e.g. batch, temperature, sequencing technology).
```
# Let’s Define these Inputs:
```{r}
# remove the 'width' column
countData <- as.matrix(subset(counts, select = c(-width)))
#define the experimental setup
colData <- read.table(coldata_file, header = T, sep = '\t', stringsAsFactors = TRUE)
#define the design formula
designFormula <- "~ group"
# Now, we are ready to run DESeq2
library(DESeq2)
library(stats)
#create a DESeq dataset object from the count matrix and the colData
dds <- DESeqDataSetFromMatrix(countData = countData,
colData = colData,
design = as.formula(designFormula))
#print dds object to see the contents
print(dds)
# Remove genes that have almost no information in any of the given samples
# For each gene, we count the total number of reads for that gene in all samples and remove those that don't have at least 1 read.
dds <- dds[ rowSums(DESeq2::counts(dds)) > 1,]
dds <- DESeq(dds)
# compute the contrast for the 'group' variable where 'CTRL'
# samples are used as the control group.
DEresults = results(dds, contrast = c("group", 'CASE', 'CTRL'))
#sort results by increasing p-value
DEresults <- DEresults[order(DEresults$pvalue),]
# Thus we have obtained a table containing the differential expression status of case samples compared to the control samples.
# shows a summary of the results
print(DEresults)
```
# Diagnostic Plots
```{r}
## before proceeding to do any downstream analysis and jumping to conclusions about biological insights that are rechable with experimental data at hand. It is important to do some more diagnostic test to improve our confidence about the quality of the data and experimental setup.
# MA plot
# An MA-plot is a plot of log-intensity ratios (M-values) versus log-intensity averages (A-values).
DESeq2::plotMA(object = dds, ylim = c(-5,5))
# p-value distribution
# observe the distribution of raw p-values
library(ggplot2)
# FIGURE: P-value distribution genes before adjusting for multiple testing.
ggplot(data = as.data.frame(DEresults), aes(x = pvalue)) + geom_histogram(bins = 100)
```
# PCA Plot
```{r}
# A final diagnosis is to check the biological reproducibility of the sample replicates in a PCA plot or a heatmap.
# To plot the PCA results, we need to extract the normalized counts from the DESeqDataSet object. It is possible to color the points in the scatter plot by the variable of interest, which helps to see if the replicates cluster well.
# extract normalized counts from the DESeqDataSet object
countsNormalized <- DESeq2::counts(dds, normalized = TRUE)
# select top 500 most variable genes
selectedGenes <- names(sort(apply(countsNormalized, 1, var),
decreasing = TRUE)[1:500])
#Figure: PCA plot of top 500 most variable genes
plotPCA(countsNormalized[selectedGenes,],
col = as.numeric(colData$group), adj = 0.5,
xlim = c(-0.5, 0.5), ylim = c(-0.5, 0.6))
Alternatively, the normalized counts can be transformed using the DESeq2::rlog
function and DESeq2::plotPCA() can be readily used to plot the PCA results 
```
