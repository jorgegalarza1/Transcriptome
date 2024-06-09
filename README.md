# Load libraries

# install.packages("BiocManager")
# BiocManager::install("DESeq2")
# BiocManager::install("apeglm")
# BiocManager::install("EnhancedVolcano")
# BiocManager::install("biomaRt")
# BiocManager::install("AnnotationHub")
library(DESeq2)
library(pheatmap)
library(dplyr)
library(RColorBrewer)
library(ggplot2)
library(tidyverse)
library(ggrepel)
library(apeglm)
library(EnhancedVolcano)
library(biomaRt)
library(ggtext)
library(clusterProfiler)
library(ggpubr)
library(AnnotationHub)
library(scales)
library(ggplotify)

# Working directory
setwd("/Users/galarza/Documents/NMSU/Chapter_4_(Transcriptome_Analysis)/Data/Transcriptome/Data_analysis")

######################################################
######################################################
######################################################
######################################################
######################################################
######################################################
#################### CF- VS CF+ ######################
######################################################
######################################################
######################################################
######################################################
######################################################
######################################################

# Entire gene data
# genome_annotation_data <- read.csv("GCF_003789085.1_ASM378908v1_genomic.csv", header = TRUE)

# Filter to get only genes
# genome_annotation_data <- genome_annotation_data %>%
  # filter(X3 == "gene")

# Export as csv
# write.csv(genome_annotation_data, "genome_annotation_data_only_genes.csv")

# Load data
CF_CF_count_data <- read.csv("CF_CF.csv", header = TRUE, row.names = 1)
colnames(CF_CF_count_data)
head(CF_CF_count_data)

CF_CF_count_data <- na.omit(CF_CF_count_data)

# Load sample information
CF_CF_sample_info <- read.csv("CF_CF_design.csv", header = TRUE, row.names = 1)
colnames(CF_CF_sample_info)
head(CF_CF_sample_info)

# Set factor levels
CF_CF_sample_info$Status <- factor(CF_CF_sample_info$Status)

# Create DESeq object and import count data and sample information
CF_CF_dds <- DESeqDataSetFromMatrix(countData = CF_CF_count_data,
                              colData = CF_CF_sample_info,
                              design = ~Status)

# Prefiltering
smallestGroupSize <- 3
CF_CF_keep <- rowSums(counts(CF_CF_dds) >= 5) >= smallestGroupSize
CF_CF_dds <- CF_CF_dds[CF_CF_keep,]

# Set reference for the treatment factor
CF_CF_dds$Status <- factor(CF_CF_dds$Status, levels = c("Unchallenged","Challenged"))

# Perform statistical test (s) to identify differential expressed genes
CF_CF_dds <- DESeq(CF_CF_dds)

CF_CF_res <- results(CF_CF_dds)
summary(CF_CF_res)

resultsNames(CF_CF_dds)

# Dispersion estimates
plotDispEsts(CF_CF_dds)

# DESeq results
CF_CF_res <- results(CF_CF_dds, alpha = 0.05)
summary(CF_CF_res)
CF_CF_res

# MA Plot
plotMA(CF_CF_res, ylim = c(-10,10))

# Normalized data
CF_CF_normalized_dds <- counts(CF_CF_dds, normalized = TRUE) 
#write.csv(CF_CF_normalized_dds, "CF_CF_normalized.csv")

# Ordered data
ordered_CF_CF <- CF_CF_res[order(CF_CF_res$padj),]
#write.csv(ordered_CF_CF, "CF_CF_ordered.csv")

# Import new CSV files
normal_data_CF_CF <- read.csv("CF_CF_normalized.csv", header = TRUE, row.names = 1)
ordered_data_CF_CF <- read.csv("CF_CF_ordered.csv", header = TRUE, row.names = 1)

top_padj_CF_CF <- ordered_data_CF_CF %>%
  filter(padj < 0.05)

#write.csv(top_padj_CF_CF, "CF_CF_filtered_sigs.csv")

# CF_CF significant up and down regulated
filtered_sigs_up_CF_CF <- ordered_data_CF_CF %>%
  filter(padj < 0.05) %>%
  filter(log2FoldChange > 1)

filtered_sigs_down_CF_CF <- ordered_data_CF_CF %>%
  filter(padj < 0.05) %>%
  filter(log2FoldChange < -1)

#write.csv(filtered_sigs_up_CF_CF, "CF_CF_filtered_sigs_up.csv")
#write.csv(filtered_sigs_down_CF_CF, "CF_CF_filtered_sigs_down.csv")

# Volcano plot
ordered_data_CF_CF$diffexpressed <- "NO"
ordered_data_CF_CF$diffexpressed[ordered_data_CF_CF$log2FoldChange > 1 & ordered_data_CF_CF$padj < 0.05] <- "UP"
ordered_data_CF_CF$diffexpressed[ordered_data_CF_CF$log2FoldChange < -1 & ordered_data_CF_CF$padj < 0.05] <- "DOWN"

CF_CF_volcano_plot <- ggplot(data = ordered_data_CF_CF, aes(x = log2FoldChange, y = -log10(padj), col = diffexpressed)) +
  geom_vline(xintercept = c(-1, 1), col = "black", linetype = 'dashed') +
  geom_hline(yintercept = -log10(0.05), col = "black", linetype = 'dashed') +
  geom_point(size = 2, alpha = 0.6) +
  scale_color_manual(values = c("#00AFBB", "darkgray", "red"), 
                     labels = c("Down-regulated", "Not significant", "Up-regulated")) +
  coord_cartesian(ylim = c(0, 8), xlim = c(-12, 12)) +
  labs(color = 'Gene Regulation',
       x = expression("log"[2]*"[FC] CF-/CF+"), y = expression("-log"[10]*"[FDR]")) + 
  scale_x_continuous(breaks = seq(-12, 12, 2)) +
  theme_classic(base_size = 14) + 
  guides(alpha = "none", color = "none")

# Heat map colors
my_colors <- colorRampPalette(c("yellow", "orange4", "midnightblue"))(50)
status_annotation_coloration <- list(Status = c(Unchallenged = "#F8766D", Challenged = "#00AFBB"))

# Annotation names
CF_CF_annot_info <- as.data.frame(colData(CF_CF_dds)[,c("Status")])

# z-scores for heat map
cal_z_score <- function(x) {(x-mean(x)) / sd(x)}

# Top 10 based on FDR (False detection rate/ adjusted p-value)
CF_CF_top_hits <- ordered_data_CF_CF[order(ordered_data_CF_CF$padj), ][1:204,]
CF_CF_top_hits <- row.names(CF_CF_top_hits)
CF_CF_top_hits

CF_CF_zscore_all <- t(apply(normal_data_CF_CF, 1, cal_z_score))
CF_CF_zscore_subset <- CF_CF_zscore_all[CF_CF_top_hits,]

CF_CF_cdata <- colData(CF_CF_dds)

CF_CF_heatmap <- pheatmap(CF_CF_zscore_subset,
         scale = "row",
         clustering_distance_rows = "euclidean",
         clustering_distance_cols = "euclidean",
         cluster_rows = TRUE,
         cluster_cols = FALSE,
         show_rownames = FALSE,
         show_colnames = FALSE,
         annotation_col = as.data.frame(CF_CF_cdata["Status"]),
         annotation_names_col = FALSE,
         annotation_colors = status_annotation_coloration,
         fontsize = 12,
         legend = FALSE)

# Variance stabilizing transformation
CF_CF_vsd <- vst(CF_CF_dds, blind = FALSE)

# PCA Plot
CF_CF_rld <- rlog(CF_CF_dds, blind = FALSE)

CF_CF_PCA_data <- plotPCA(CF_CF_rld, intgroup = "Status", returnData = TRUE)

CF_CF_PCA_plot <- ggplot(CF_CF_PCA_data, aes(x = PC1, y = PC2, color = Status)) +
  geom_point(size = 3) +
  geom_text(aes(label = name), vjust = -1, show.legend = FALSE, size = 6) +
  ylim(-40,40) +
  xlim(-40, 40) +
  theme_classic(base_size = 14) +
  theme(legend.position="left")

# Heatmap (generate distance matrix and choose colors)
CF_CF_sampleDists <- dist(t(assay(CF_CF_vsd)))
CF_CF_sampleDistsMatrix <- as.matrix(CF_CF_sampleDists)
colnames(CF_CF_sampleDistsMatrix)

colors <- colorRampPalette(rev(brewer.pal(9, "Blues")))(5)

CF_CF_distance_matrix <- pheatmap(CF_CF_sampleDistsMatrix,
         clustering_distance_rows = CF_CF_sampleDists,
         clustering_distance_cols = CF_CF_sampleDists,
         col = colors,
         cex = 1.1)

CF_CF_volcano_plot
CF_CF_heatmap
CF_CF_PCA_plot
CF_CF_distance_matrix

######################################################
######################################################
######################################################
######################################################
######################################################
######################################################
#################### PBF- VS PBF+ ####################
######################################################
######################################################
######################################################
######################################################
######################################################
######################################################

# Load data
PBF_PBF_count_data <- read.csv("PBF_PBF.csv", header = TRUE, row.names = 1)
colnames(PBF_PBF_count_data)
head(PBF_PBF_count_data)

PBF_PBF_count_data <- na.omit(PBF_PBF_count_data)

# Load sample information
PBF_PBF_sample_info <- read.csv("PBF_PBF_design.csv", header = TRUE, row.names = 1)
colnames(PBF_PBF_sample_info)
head(PBF_PBF_sample_info)

# Set factor levels
PBF_PBF_sample_info$Status <- factor(PBF_PBF_sample_info$Status)

# Create DESeq object and import count data and sample information
PBF_PBF_dds <- DESeqDataSetFromMatrix(countData = PBF_PBF_count_data,
                              colData = PBF_PBF_sample_info,
                              design = ~Status)

# Prefiltering
PBF_PBF_keep <- rowSums(counts(PBF_PBF_dds) >= 5) >= smallestGroupSize
PBF_PBF_dds <- PBF_PBF_dds[PBF_PBF_keep,]

# Set reference for the treatment factor
PBF_PBF_dds$Status <- factor(PBF_PBF_dds$Status, levels = c("Unchallenged","Challenged"))

# Perform statistical test (s) to identify differential expressed genes
PBF_PBF_dds <- DESeq(PBF_PBF_dds)
resultsNames(PBF_PBF_dds)

# Dispersion estimates
plotDispEsts(PBF_PBF_dds)

# DESeq results
PBF_PBF_res <- results(PBF_PBF_dds, alpha = 0.05)
summary(PBF_PBF_res)

# MA Plot
plotMA(PBF_PBF_res, ylim = c(-10,10))

# Normalized data
PBF_PBF_normalized_dds <- counts(PBF_PBF_dds, normalized = TRUE) 
#write.csv(PBF_PBF_normalized_dds, "PBF_PBF_normalized.csv")

# Ordered data
ordered_PBF_PBF <- PBF_PBF_res[order(PBF_PBF_res$padj),]
#write.csv(ordered_PBF_PBF, "PBF_PBF_ordered.csv")

# Import new CSV files
normal_data_PBF_PBF <- read.csv("PBF_PBF_normalized.csv", header = TRUE, row.names = 1)
ordered_data_PBF_PBF <- read.csv("PBF_PBF_ordered.csv", header = TRUE, row.names = 1)

top_padj_PBF_PBF <- ordered_data_PBF_PBF %>%
  filter(padj < 0.05)

#write.csv(top_padj_PBF_PBF, "PBF_PBF_filtered_sigs.csv")

# PBF_PBF significant up and down regulated
filtered_sigs_up_PBF_PBF <- ordered_data_PBF_PBF %>%
  filter(padj < 0.05) %>%
  filter(log2FoldChange >1)

filtered_sigs_down_PBF_PBF <- ordered_data_PBF_PBF %>%
  filter(padj < 0.05) %>%
  filter(log2FoldChange < -1)

#write.csv(filtered_sigs_up_PBF_PBF, "PBF_PBF_filtered_sigs_up.csv")
#write.csv(filtered_sigs_down_PBF_PBF, "PBF_PBF_filtered_sigs_down.csv")

# Volcano plot
ordered_data_PBF_PBF$diffexpressed <- "NO"
ordered_data_PBF_PBF$diffexpressed[ordered_data_PBF_PBF$log2FoldChange > 1 & ordered_data_PBF_PBF$padj < 0.05] <- "UP"
ordered_data_PBF_PBF$diffexpressed[ordered_data_PBF_PBF$log2FoldChange < -1 & ordered_data_PBF_PBF$padj < 0.05] <- "DOWN"

PBF_PBF_volcano_plot <- ggplot(data = ordered_data_PBF_PBF, aes(x = log2FoldChange, y = -log10(padj), col = diffexpressed)) +
  geom_vline(xintercept = c(-1, 1), col = "black", linetype = 'dashed') +
  geom_hline(yintercept = -log10(0.05), col = "black", linetype = 'dashed') +
  geom_point(size = 2, alpha = 0.6) +
  scale_color_manual(values = c("#00AFBB", "darkgray", "red"), 
                     labels = c("Down-regulated", "Not significant", "Up-regulated")) +
  coord_cartesian(ylim = c(0, 8), xlim = c(-12, 12)) +
  labs(color = 'Gene Regulation',
       x = expression("log"[2]*"[FC] PBF-/PBF+"), y = expression("-log"[10]*"[FDR]")) + 
  scale_x_continuous(breaks = seq(-12, 12, 2)) +
  theme_classic(base_size = 14) + 
  guides(alpha = "none", color = "none")

# Annotation names
PBF_PBF_annot_info <- as.data.frame(colData(PBF_PBF_dds)[,c("Status")])

# Top 10 based on FDR (False detection rate/ adjusted p-value)
PBF_PBF_top_hits <- ordered_data_PBF_PBF[order(ordered_data_PBF_PBF$padj), ][1:42,]
PBF_PBF_top_hits <- row.names(PBF_PBF_top_hits)
PBF_PBF_top_hits

PBF_PBF_zscore_all <- t(apply(normal_data_PBF_PBF, 1, cal_z_score))
PBF_PBF_zscore_subset <- PBF_PBF_zscore_all[PBF_PBF_top_hits,]

PBF_PBF_cdata <- colData(PBF_PBF_dds)

PBF_PBF_heatmap <- pheatmap(PBF_PBF_zscore_subset,
         scale = "row",
         clustering_distance_rows = "euclidean",
         clustering_distance_cols = "euclidean",
         cluster_rows = TRUE,
         cluster_cols = FALSE,
         show_rownames = FALSE,
         show_colnames = FALSE,
         annotation_col = as.data.frame(PBF_PBF_cdata["Status"]),
         annotation_names_col = FALSE,
         annotation_colors = status_annotation_coloration,
         fontsize = 12,
         legend = TRUE)

# Variance stabilizing transformation
PBF_PBF_vsd <- vst(PBF_PBF_dds, blind = FALSE)

# PCA Plot
PBF_PBF_rld <- rlog(PBF_PBF_dds, blind = FALSE)

PBF_PBF_PCA_data <- plotPCA(PBF_PBF_rld, intgroup = "Status", returnData = TRUE)

PBF_PBF_PCA_plot <- ggplot(PBF_PBF_PCA_data, aes(x = PC1, y = PC2, color = Status)) +
  geom_point(size = 3) +
  geom_text(aes(label = name), vjust = -1, show.legend = FALSE, size = 6) +
  ylim(-30,30) +
  xlim(-50, 50) +
  theme_classic(base_size = 20) +
  theme(legend.position="bottom")

# Heatmap (generate distance matrix and choose colors)
PBF_PBF_sampleDists <- dist(t(assay(PBF_PBF_vsd)))
PBF_PBF_sampleDistsMatrix <- as.matrix(PBF_PBF_sampleDists)
colnames(PBF_PBF_sampleDistsMatrix)

PBF_PBF_distance_matrix <- pheatmap(PBF_PBF_sampleDistsMatrix,
         clustering_distance_rows = PBF_PBF_sampleDists,
         clustering_distance_cols = PBF_PBF_sampleDists,
         col = colors,
         cex = 1.1)

PBF_PBF_volcano_plot
PBF_PBF_heatmap
PBF_PBF_PCA_plot
PBF_PBF_distance_matrix

######################################################
######################################################
######################################################
######################################################
######################################################
######################################################
################# Positive Status ####################
######################################################
######################################################
######################################################
######################################################
######################################################
######################################################

# Load data
PS_count_data <- read.csv("Positive_status.csv", header = TRUE, row.names = 1)
colnames(PS_count_data)
head(PS_count_data)

PS_count_data <- na.omit(PS_count_data)

# Load sample information
PS_sample_info <- read.csv("Positive_status_design.csv", header = TRUE, row.names = 1)
colnames(PS_sample_info)
head(PS_sample_info)

# Set factor levels
PS_sample_info$Treatment <- factor(PS_sample_info$Treatment)

# Create DESeq object and import count data and sample information
PS_dds <- DESeqDataSetFromMatrix(countData = PS_count_data,
                                      colData = PS_sample_info,
                                      design = ~Treatment)

# Prefiltering
PS_keep <- rowSums(counts(PS_dds) >= 5) >= smallestGroupSize
PS_dds <- PS_dds[PS_keep,]

# Set reference for the treatment factor
PS_dds$Treatment <- factor(PS_dds$Treatment, levels = c("CF","PBF"))

# Perform statistical test (s) to identify differential expressed genes
PS_dds <- DESeq(PS_dds)
resultsNames(PS_dds)

# Dispersion estimates
plotDispEsts(PS_dds)

# DESeq results
PS_res <- results(PS_dds, alpha = 0.05)
summary(PS_res)

# MA Plot
plotMA(PS_res, ylim = c(-10,10))

# Normalized data
PS_normalized <- counts(PS_dds, normalized = TRUE) 
#write.csv(PS_normalized, "PS_normalized.csv")

# Ordered data
ordered_positive_status <- PS_res[order(PS_res$padj),]
#write.csv(ordered_positive_status, "PS_ordered.csv")

# Import new CSV files
normal_data_positive_status <- read.csv("PS_normalized.csv", header = TRUE, row.names = 1)
ordered_data_positive_status <- read.csv("PS_ordered.csv", header = TRUE, row.names = 1)

top_padj_positive_status <- ordered_data_positive_status %>%
  filter(padj < 0.05)

#write.csv(top_padj_positive_status, "PS_filtered_sigs.csv")

# Positive_status significant up and down regulated
filtered_sigs_up_positive_status <- ordered_data_positive_status %>%
  filter(padj < 0.05) %>%
  filter(log2FoldChange >1)

filtered_sigs_down_positive_status <- ordered_data_positive_status %>%
  filter(padj < 0.05) %>%
  filter(log2FoldChange < -1)

#write.csv(filtered_sigs_up_positive_status, "PS_filtered_sigs_up.csv")
#write.csv(filtered_sigs_down_positive_status, "PS_filtered_sigs_down.csv")

# Volcano plot
ordered_data_positive_status$diffexpressed <- "NO"
ordered_data_positive_status$diffexpressed[ordered_data_positive_status$log2FoldChange > 1 & ordered_data_positive_status$padj < 0.05] <- "UP"
ordered_data_positive_status$diffexpressed[ordered_data_positive_status$log2FoldChange < -1 & ordered_data_positive_status$padj < 0.05] <- "DOWN"

PS_volcano_plot <- ggplot(data = ordered_data_positive_status, aes(x = log2FoldChange, y = -log10(padj), col = diffexpressed)) +
  geom_vline(xintercept = c(-1, 1), col = "black", linetype = 'dashed') +
  geom_hline(yintercept = -log10(0.05), col = "black", linetype = 'dashed') +
  geom_point(size = 2, alpha = 0.6) +
  scale_color_manual(values = c("#00AFBB", "darkgray", "red"), 
                     labels = c("Down-regulated", "Not significant", "Up-regulated")) +
  coord_cartesian(ylim = c(0, 8), xlim = c(-12, 12)) +
  labs(color = 'Gene Regulation',
       x = expression("log"[2]*"[FC] CF+/PBF+"), y = expression("-log"[10]*"[FDR]")) + 
  scale_x_continuous(breaks = seq(-12, 12, 2)) +
  theme_classic(base_size = 14) + 
  guides(alpha = "none", color = "none")

# Heat map colors
treatment_annotation_coloration <- list(Treatment = c(CF = "#F8766D", PBF = "#00AFBB"))

# Annotation names
PS_annot_info <- as.data.frame(colData(PS_dds)[,c("Treatment")])

# Top 10 based on FDR (False detection rate/ adjusted p-value)
PS_top_hits <- ordered_data_positive_status[order(ordered_data_positive_status$padj), ][1:37,]
PS_top_hits <- row.names(PS_top_hits)
PS_top_hits

PS_zscore_all <- t(apply(normal_data_positive_status, 1, cal_z_score))
PS_zscore_subset <- PS_zscore_all[PS_top_hits,]

PS_cdata <- colData(PS_dds)

PS_heatmap <- pheatmap(PS_zscore_subset,
         scale = "row",
         clustering_distance_rows = "euclidean",
         clustering_distance_cols = "euclidean",
         cluster_rows = TRUE,
         cluster_cols = FALSE,
         show_rownames = FALSE,
         show_colnames = FALSE,
         annotation_col = as.data.frame(PS_cdata["Treatment"]),
         annotation_names_col = FALSE,
         annotation_colors = treatment_annotation_coloration,
         fontsize = 12,
         legend = TRUE)

# Variance stabilizing transformation
PS_vsd <- vst(PS_dds, blind = FALSE)

# PCA Plot
PS_rld <- rlog(PS_dds, blind = FALSE)

PS_PCA_data <- plotPCA(PS_rld, intgroup = "Treatment", returnData = TRUE)

PS_PCA_plot <- ggplot(PS_PCA_data, aes(x = PC1, y = PC2, color = Treatment)) +
  geom_point(size = 3) +
  geom_text(aes(label = name), vjust = -1, show.legend = FALSE, size = 6) +
  ylim(-30,30) +
  xlim(-50, 50) +
  theme_classic(base_size = 20) +
  theme(legend.position="bottom")

# Heatmap (generate distance matrix and choose colors)
PS_sampleDists <- dist(t(assay(PS_vsd)))
PS_sampleDistsMatrix <- as.matrix(PS_sampleDists)
colnames(PS_sampleDistsMatrix)

PS_distance_matrix <- pheatmap(PS_sampleDistsMatrix,
         clustering_distance_rows = PS_sampleDists,
         clustering_distance_cols = PS_sampleDists,
         col = colors,
         cex = 1.1)

PS_volcano_plot
PS_heatmap
PS_PCA_plot
PS_distance_matrix

######################################################
######################################################
######################################################
######################################################
######################################################
######################################################
################# Negative Status ####################
######################################################
######################################################
######################################################
######################################################
######################################################
######################################################

# Load data
NS_count_data <- read.csv("Negative_status.csv", header = TRUE, row.names = 1)
colnames(NS_count_data)
head(NS_count_data)

NS_count_data <- na.omit(NS_count_data)

# Load sample information
NS_sample_info <- read.csv("Negative_status_design.csv", header = TRUE, row.names = 1)
colnames(NS_sample_info)
head(NS_sample_info)

# Set factor levels
NS_sample_info$Treatment <- factor(NS_sample_info$Treatment)

# Create DESeq object and import count data and sample information
NS_dds <- DESeqDataSetFromMatrix(countData = NS_count_data,
                                 colData = NS_sample_info,
                                 design = ~Treatment)

# Prefiltering
NS_keep <- rowSums(counts(NS_dds) >= 5) >= smallestGroupSize
NS_dds <- NS_dds[NS_keep,]

# Set reference for the treatment factor
NS_dds$Treatment <- factor(NS_dds$Treatment, levels = c("CF","PBF"))

# Perform statistical test (s) to identify differential expressed genes
NS_dds <- DESeq(NS_dds)
resultsNames(NS_dds)

# Dispersion estimates
plotDispEsts(NS_dds)

# DESeq results
NS_res <- results(NS_dds, alpha = 0.05)
summary(NS_res)

# MA Plot
plotMA(NS_res, ylim = c(-10,10))

# Normalized data
NS_normalized_dds <- counts(NS_dds, normalized = TRUE) 
#write.csv(NS_normalized_dds, "NS_normalized.csv")

# Ordered data
ordered_negative_status <- NS_res[order(NS_res$padj),]
#write.csv(ordered_negative_status, "NS_ordered_MOD.csv")

# Import new CSV files
normal_data_negative_status <- read.csv("NS_normalized.csv", header = TRUE, row.names = 1)
ordered_data_negative_status <- read.csv("NS_ordered.csv", header = TRUE, row.names = 1)

top_padj_negative_status <- ordered_data_negative_status %>%
  filter(padj < 0.05)

#write.csv(top_padj_negative_status, "NS_filtered_sigs.csv")

# Negative_status significant up and down regulated
filtered_sigs_up_negative_status <- ordered_data_negative_status %>%
  filter(padj < 0.05) %>%
  filter(log2FoldChange >1)

filtered_sigs_down_negative_status <- ordered_data_negative_status %>%
  filter(padj < 0.05) %>%
  filter(log2FoldChange < -1)

#write.csv(filtered_sigs_up_negative_status, "NS_filtered_sigs_up.csv")
#write.csv(filtered_sigs_down_negative_status, "NS_filtered_sigs_down.csv")

# Volcano plot
ordered_data_negative_status$diffexpressed <- "NO"
ordered_data_negative_status$diffexpressed[ordered_data_negative_status$log2FoldChange > 1 & ordered_data_negative_status$padj < 0.05] <- "UP"
ordered_data_negative_status$diffexpressed[ordered_data_negative_status$log2FoldChange < -1 & ordered_data_negative_status$padj < 0.05] <- "DOWN"

NS_volcano_plot <- ggplot(data = ordered_data_negative_status, aes(x = log2FoldChange, y = -log10(padj), col = diffexpressed)) +
  geom_vline(xintercept = c(-1, 1), col = "black", linetype = 'dashed') +
  geom_hline(yintercept = -log10(0.05), col = "black", linetype = 'dashed') +
  geom_point(size = 2, alpha = 0.6) +
  scale_color_manual(values = c("#00AFBB", "darkgray", "red"), 
                     labels = c("Down-regulated", "Not significant", "Up-regulated")) +
  coord_cartesian(ylim = c(0, 8), xlim = c(-12, 12)) +
  labs(color = 'Gene Regulation',
       x = expression("log"[2]*"[FC] CF-/PBF-"), y = expression("-log"[10]*"[FDR]")) + 
  scale_x_continuous(breaks = seq(-12, 12, 2)) +
  theme_classic(base_size = 14) + 
  guides(alpha = "none", color = "none")

# Annotation names
NS_annot_info <- as.data.frame(colData(NS_dds)[,c("Treatment")])

# Top 10 based on FDR (False detection rate/ adjusted p-value)
NS_top_hits <- ordered_data_negative_status[order(ordered_data_negative_status$padj), ][1:204,]
NS_top_hits <- row.names(NS_top_hits)
NS_top_hits

NS_zscore_all <- t(apply(normal_data_negative_status, 1, cal_z_score))
NS_zscore_subset <- NS_zscore_all[NS_top_hits,]

NS_cdata <- colData(NS_dds)

NS_heatmap <- pheatmap(NS_zscore_subset,
         scale = "row",
         clustering_distance_rows = "euclidean",
         clustering_distance_cols = "euclidean",
         cluster_rows = TRUE,
         cluster_cols = FALSE,
         show_rownames = FALSE,
         show_colnames = FALSE,
         annotation_col = as.data.frame(NS_cdata["Treatment"]),
         annotation_names_col = FALSE,
         annotation_colors = treatment_annotation_coloration,
         fontsize = 12,
         legend = TRUE)

# Variance stabilizing transformation
NS_vsd <- vst(NS_dds, blind = FALSE)

# PCA Plot
NS_rld <- rlog(NS_dds, blind = FALSE)

NS_PCA_data <- plotPCA(NS_rld, intgroup = "Treatment", returnData = TRUE)

NS_PCA_plot <- ggplot(NS_PCA_data, aes(x = PC1, y = PC2, color = Treatment)) +
  geom_point(size = 3) +
  geom_text(aes(label = name), vjust = -1, show.legend = FALSE, size = 6) +
  ylim(-30,30) +
  xlim(-50, 50) +
  theme_classic(base_size = 20) +
  theme(legend.position="bottom")

# Heatmap (generate distance matrix and choose colors)
NS_sampleDists <- dist(t(assay(NS_vsd)))
NS_sampleDistsMatrix <- as.matrix(NS_sampleDists)
colnames(NS_sampleDistsMatrix)

NS_distance_matrix <- pheatmap(NS_sampleDistsMatrix,
         clustering_distance_rows = NS_sampleDists,
         clustering_distance_cols = NS_sampleDists,
         col = colors,
         cex = 1.1)

NS_volcano_plot
NS_heatmap
NS_PCA_plot
NS_distance_matrix

######################################################
######################################################
######################################################
######################################################
######################################################
######################################################
#################### Gene Ontology ###################
######################################################
######################################################
######################################################
######################################################
######################################################
######################################################
# CF_UP <- read.csv("GO/CF_UP.csv", header = TRUE)
# CF_DOWN <- read.csv("GO/CF_DOWN.csv", header = TRUE)
# CF_UP_DOWN <- read.csv("GO/CF_UP_DOWN.csv", header = TRUE)

# PBF_UP <- read.csv("GO/PBF_UP.csv",header = TRUE)
# PBF_DOWN <- read.csv("GO/PBF_DOWN.csv",header = TRUE)
# PBF_UP_DOWN <- read.csv("GO/PBF_UP_DOWN.csv",header = TRUE)
# 
# PS_UP <- read.csv("GO/PS_UP.csv",header = TRUE)
# PS_DOWN <- read.csv("GO/PS_DOWN.csv",header = TRUE)
# PS_UP_DOWN <- read.csv("GO/PS_UP_DOWN.csv",header = TRUE)
# 
# NS_UP <- read.csv("GO/NS_UP.csv",header = TRUE)
# NS_DOWN <- read.csv("GO/NS_DOWN.csv",header = TRUE)
# NS_UP_DOWN <- read.csv("GO/NS_UP_DOWN.csv",header = TRUE)

###################################################### CF_UP
# CF_UP <- CF_UP %>%
#   filter(adjusted_p_value < 0.05) %>%
#   arrange(adjusted_p_value) %>%
#   top_n(20)
# 
# #write.csv(CF_UP,"CF_UP_richfactor.csv")
# 
# CF_UP <- CF_UP %>%
#   mutate(RichFactor = intersection_size/query_size)
# 
# CF_UP$append <- paste(CF_UP$term_name,CF_UP$term_id)

#write.csv(CF_UP, "CF_up_final.csv")
CF_UP <- read.csv("GO/CF_up_final.csv", header = TRUE)

CF_UP_GO <- ggplot(data = CF_UP, aes(x = append, y = RichFactor,
                         size = intersection_size,
                         color = adjusted_p_value)) +
  geom_point(aes(reorder(append, -adjusted_p_value, y=RichFactor))) +
  coord_flip() +
  theme_linedraw(base_size = 12) +
  labs(x = "GOterm", size="GeneNumber", color="qValue") +
  scale_color_gradient(low="red", high="blue") +
  scale_size_continuous(limits = c(1, 30),
                        range = c(1,5)) +
  scale_y_continuous(labels = label_number(accuracy = 0.01)) +
  theme(axis.text.x = element_text(angle = 45,
                                   hjust = 1)) +
  facet_wrap(~source)

###################################################### PBF_UP
# 
# PBF_UP <- PBF_UP %>%
#   filter(adjusted_p_value < 0.05) %>%
#   arrange(adjusted_p_value) %>%
#   top_n(20)
# 
# #write.csv(PBF_UP,"PBF_UP_richfactor.csv")
# 
# PBF_UP <- PBF_UP %>%
#   mutate(RichFactor = intersection_size/query_size)
# 
# PBF_UP$append <- paste(PBF_UP$term_name,PBF_UP$term_id)
# 
# write.csv(PBF_UP, "PBF_up_final.csv")
PBF_UP <- read.csv("GO/PBF_up_final.csv", header = TRUE)

PBF_UP_GO <- ggplot(data = PBF_UP, aes(x = append, y = RichFactor,
                          size = intersection_size,
                          color = adjusted_p_value)) +
  geom_point(aes(reorder(append, -adjusted_p_value), y=RichFactor)) +
  coord_flip() +
  theme_linedraw(base_size = 12) +
  labs(x = "GOterm", size="GeneNumber     ", color="qValue") +
  scale_color_gradient(low="red", high="blue") +
  scale_size_continuous(limits = c(1, 20),
                        range = c(1,5)) +
  scale_y_continuous(labels = label_number(accuracy = 0.01)) +
  theme(axis.text.x = element_text(angle = 45,
                                   hjust = 1)) +
  facet_wrap(~source)


###################################################### CF_DOWN

# CF_DOWN <- CF_DOWN %>%
#   filter(adjusted_p_value < 0.05) %>%
#   arrange(adjusted_p_value) %>%
#   top_n(20)
# 
# #write.csv(CF_DOWN,"CF_DOWN_richfactor.csv")
# 
# CF_DOWN <- CF_DOWN %>%
#   mutate(RichFactor = intersection_size/query_size)
# 
# CF_DOWN$append <- paste(CF_DOWN$term_name,CF_DOWN$term_id)
# 
# write.csv(CF_DOWN, "CF_down_final.csv")
CF_DOWN <- read.csv("GO/CF_down_final.csv", header = TRUE)

CF_DOWN_GO <- ggplot(data = CF_DOWN, aes(x = append, y = RichFactor,
                           size = intersection_size,
                           color = adjusted_p_value)) +
  geom_point(aes(reorder(append, -adjusted_p_value), y=RichFactor)) +
  coord_flip() +
  theme_linedraw(base_size = 12) +
  labs(x = "GOterm", size="GeneNumber     ", color="qValue") +
  scale_color_gradient(low="red", high="blue") +
  scale_size_continuous(limits = c(1, 40),
                        range = c(1,5)) +
  scale_y_continuous(labels = label_number(accuracy = 0.01)) +
  theme(axis.text.x = element_text(angle = 45,
                                   hjust = 1)) +
  facet_wrap(~source)

###################################################### PBF_DOWN

# PBF_DOWN <- PBF_DOWN %>%
#   filter(adjusted_p_value < 0.05) %>%
#   arrange(adjusted_p_value) %>%
#   top_n(20)
# 
# #write.csv(PBF_DOWN,"PBF_DOWN_richfactor.csv")
# 
# PBF_DOWN <- PBF_DOWN %>%
#   mutate(RichFactor = intersection_size/query_size)
# 
# PBF_DOWN$append <- paste(PBF_DOWN$term_name,PBF_DOWN$term_id)
# 
# write.csv(PBF_DOWN, "PBF_down_final.csv")
PBF_DOWN <- read.csv("GO/PBF_down_final.csv", header = TRUE)

PBF_DOWN_GO <- ggplot(data = PBF_DOWN, aes(x = append, y = RichFactor,
                            size = intersection_size,
                            color = adjusted_p_value)) +
  geom_point(aes(reorder(append, -adjusted_p_value), y=RichFactor)) +
  coord_flip() +
  theme_linedraw(base_size = 12) +
  labs(x = "GOterm", size="GeneNumber ", color="qValue") +
  scale_color_gradient(low="red", high="blue") +
  scale_size_continuous(limits = c(1, 4),
                        range = c(1,5)) +
  scale_y_continuous(labels = label_number(accuracy = 0.01)) +
  theme(axis.text.x = element_text(angle = 45,
                                   hjust = 1)) +
  facet_wrap(~source)


###################################################### PS_UP
# PS_UP <- PS_UP %>%
#   filter(adjusted_p_value < 0.05) %>%
#   arrange(adjusted_p_value) %>%
#   top_n(20)
# 
# #write.csv(PS_UP,"PS_UP_richfactor.csv")
# 
# PS_UP <- PS_UP %>%
#   mutate(RichFactor = intersection_size/4)
# 
# PS_UP$append <- paste(PS_UP$term_name,PS_UP$term_id)
# 
# write.csv(PS_UP, "PS_up_final.csv")
PS_UP <- read.csv("GO/PS_up_final.csv", header = TRUE)

PS_UP_GO <- ggplot(data = PS_UP, aes(x = append, y = RichFactor,
                         size = intersection_size,
                         color = adjusted_p_value)) +
  geom_point(aes(reorder(append, -adjusted_p_value), y=RichFactor)) +
  coord_flip() +
  theme_linedraw(base_size = 12) +
  labs(x = "GOterm", size="GeneNumber", color="qValue ") +
  scale_color_gradient(low="red", high="blue") +
  scale_size_continuous(limits = c(1, 5),
                        range = c(1,5)) +
  scale_y_continuous(labels = label_number(accuracy = 0.01)) +
  theme(axis.text.x = element_text(angle = 45,
                                   hjust = 1)) +
  facet_wrap(~source)

###################################################### NS_UP
# NS_UP <- NS_UP %>%
#   filter(adjusted_p_value < 0.05) %>%
#   arrange(adjusted_p_value) %>%
#   top_n(20)
# 
# #write.csv(NS_UP,"NS_UP_richfactor.csv")
# 
# NS_UP <- NS_UP %>%
#   mutate(RichFactor = intersection_size/372)
# 
# NS_UP$append <- paste(NS_UP$term_name,NS_UP$term_id)
# 
# write.csv(NS_UP, "NS_up_final.csv")
NS_UP <- read.csv("GO/NS_up_final.csv", header = TRUE)

NS_UP_GO <- ggplot(data = NS_UP, aes(x = append, y = RichFactor,
                         size = intersection_size,
                         color = adjusted_p_value)) +
  geom_point(aes(reorder(append, -adjusted_p_value), y=RichFactor)) +
  coord_flip() +
  theme_linedraw(base_size = 12) +
  labs(x = "GOterm", size="GeneNumber", color="qValue        ") +
  scale_color_gradient(low="red", high="blue") +
  scale_size_continuous(limits = c(1, 5),
                        range = c(1,5)) +
  scale_y_continuous(labels = label_number(accuracy = 0.01)) +
  theme(axis.text.x = element_text(angle = 45,
                                   hjust = 1)) +
  facet_wrap(~source)


###################################################### PS_DOWN
# PS_DOWN <- PS_DOWN %>%
#   filter(adjusted_p_value < 0.05) %>%
#   arrange(adjusted_p_value) %>%
#   top_n(20)
# 
# # Force top 20 selection
# PS_DOWN <- head(PS_DOWN, 20)
# 
# #write.csv(PS_DOWN,"PS_DOWN_richfactor.csv")
# 
# PS_DOWN <- PS_DOWN %>%
#   mutate(RichFactor = intersection_size/43)
# 
# PS_DOWN$append <- paste(PS_DOWN$term_name,PS_DOWN$term_id)
# 
# write.csv(PS_DOWN, "PS_down_final.csv")
PS_DOWN <- read.csv("GO/PS_down_final.csv", header = TRUE)

PS_DOWN_GO <- ggplot(data = PS_DOWN, aes(x = append, y = RichFactor,
                           size = intersection_size,
                           color = adjusted_p_value)) +
  geom_point(aes(reorder(append, -adjusted_p_value), y=RichFactor)) +
  coord_flip() +
  theme_linedraw(base_size = 12) +
  labs(x = "GOterm", size="GeneNumber", color="qValue    ") +
  scale_color_gradient(low="red", high="blue") +
  scale_size_continuous(limits = c(1, 3),
                        range = c(1,5)) +
  scale_y_continuous(labels = label_number(accuracy = 0.01)) +
  theme(axis.text.x = element_text(angle = 45,
                                   hjust = 1)) +
  facet_wrap(~source) +
  ylim(c(0.48,0.52))

###################################################### NS_DOWN
# NS_DOWN <- NS_DOWN %>%
#   filter(adjusted_p_value < 0.05) %>%
#   arrange(adjusted_p_value) %>%
#   top_n(20)
# 
# #write.csv(NS_DOWN,"NS_DOWN_richfactor.csv")
# 
# NS_DOWN <- NS_DOWN %>%
#   mutate(RichFactor = intersection_size/27)
# 
# NS_DOWN$append <- paste(NS_DOWN$term_name,NS_DOWN$term_id)
# 
# write.csv(NS_DOWN, "NS_down_final.csv")
NS_DOWN <- read.csv("GO/NS_down_final.csv", header = TRUE)

NS_DOWN_GO <- ggplot(data = NS_DOWN, aes(x = append, y = RichFactor,
                           size = intersection_size,
                           color = adjusted_p_value)) +
  geom_point(aes(reorder(append, -adjusted_p_value), y=RichFactor)) +
  coord_flip() +
  theme_linedraw(base_size = 12) +
  labs(x = "GOterm", size="GeneNumber", color="qValue   ") +
  scale_color_gradient(low="red", high="blue") +
  scale_size_continuous(limits = c(1, 100),
                        range = c(1,5)) +
  scale_y_continuous(labels = label_number(accuracy = 0.01)) +
  theme(axis.text.x = element_text(angle = 45,
                                   hjust = 1)) +
  facet_wrap(~source)


###################################################### EXPERIMENTAL

# hub <- AnnotationHub()
# query(hub, c("penaeus vannamei", "orgdb"))
# shrimp <- hub[["AH114916"]]
# new_shrimp <- hub[["AH114918"]]
# columns(shrimp)
# 
# CF_UP_enrich_GO <- enrichGO(gene = CF_UP$term_id,
#                             keyType = "GO",
#                             OrgDb = shrimp,
#                             ont = "MF",
#                             pvalueCutoff = 1,
#                             qvalueCutoff = 1,
#                             pAdjustMethod = "BH",
#                             readable = TRUE)
# 
# as.data.frame(CF_UP_enrich_GO)
# barplot(CF_UP_enrich_GO, showCategory = 10)

######################################################
######################################################
######################################################
######################################################
######################################################
######################################################
############### KEGG Path Enrichment #################
######################################################
######################################################
######################################################
######################################################
######################################################
######################################################
vannamei_db <- read.csv("vannamei_db.csv", header = TRUE)
search_kegg_organism("pvm", by = "kegg_code")

vannamei <- search_kegg_organism("Penaeus vannamei", by = "scientific_name")
dim(vannamei)
head(vannamei)

###################################################### CF

# CF_down_geneList <- read.csv("KEGG/KEGG_CF_down.csv", header = TRUE, row.names = 1)
# CF_down_geneList <- row.names(CF_down_geneList)
# CF_down_geneList
# 
# CF_down <- enrichKEGG(gene = CF_down_geneList,
#                       organism = "pvm",
#                       keyType = "ncbi-geneid",
#                       universe = unique(vannamei_db$Geneid),
#                       pAdjustMethod = "BH",
#                       use_internal_data = FALSE,
#                       pvalueCutoff = 1)
# 
# dotplot(CF_down, showCategory = 10)
# 
# #write.csv(CF_down, "KEGG/CF_DOWN_KEGG.csv")
# 
# CF_up_geneList <- read.csv("KEGG/KEGG_CF_up.csv", header = TRUE, row.names = 1)
# CF_up_geneList <- row.names(CF_up_geneList)
# CF_up_geneList
# 
# CF_up <- enrichKEGG(gene = CF_up_geneList,
#                     organism = "pvm",
#                     keyType = "ncbi-geneid",
#                     universe = unique(vannamei_db$Geneid),
#                     pAdjustMethod = "BH",
#                     use_internal_data = FALSE,
#                     pvalueCutoff = 1)
# 
# dotplot(CF_up, showCategory = 10)

#write.csv(CF_up, "KEGG/CF_UP_KEGG.csv")

CF_KEGG <- read.csv("KEGG/CF_KEGG.csv", header = TRUE)

CF_KEGG_plot <- ggplot(data = CF_KEGG, aes(x = Description, y = RichFactor,
                           size = Count,
                           color = qvalue)) +
  geom_point(aes(reorder(Description, -qvalue), y=RichFactor)) +
  coord_flip() +
  theme_linedraw(base_size = 12) +
  labs(x = "Pathway", color="qValue", size="GeneNumber") +
  scale_color_gradient(low="red", high="blue") +
  facet_wrap(~Regulation) +
  scale_y_continuous(labels = label_number(accuracy = 0.01)) +
  theme(axis.text.x = element_text(angle = 45,
                                   hjust = 1))

###################################################### PBF

# PBF_down_geneList <- read.csv("KEGG/KEGG_PBF_down.csv", header = TRUE, row.names = 1)
# PBF_down_geneList <- row.names(PBF_down_geneList)
# PBF_down_geneList
# 
# PBF_down <- enrichKEGG(gene = PBF_down_geneList,
#                        organism = "pvm",
#                        keyType = "ncbi-geneid",
#                        universe = unique(vannamei_db$Geneid),
#                        pAdjustMethod = "BH",
#                        use_internal_data = FALSE,
#                        pvalueCutoff = 1)
# 
# #write.csv(PBF_down, "KEGG/PBF_DOWN_KEGG.csv")
# 
# dotplot(PBF_down, showCategory = 10)
# 
# PBF_up_geneList <- read.csv("KEGG/KEGG_PBF_up.csv", header = TRUE, row.names = 1)
# PBF_up_geneList <- row.names(PBF_up_geneList)
# PBF_up_geneList
# 
# PBF_up <- enrichKEGG(gene = PBF_up_geneList,
#                      organism = "pvm",
#                      keyType = "ncbi-geneid",
#                      universe = unique(vannamei_db$Geneid),
#                      pAdjustMethod = "BH",
#                      use_internal_data = FALSE,
#                      pvalueCutoff = 1)
# 
# #write.csv(PBF_up, "KEGG/PBF_UP_KEGG.csv")
# 
# dotplot(PBF_up, showCategory = 10)

PBF_KEGG <- read.csv("KEGG/PBF_KEGG.csv", header = TRUE)

PBF_KEGG_plot <- ggplot(data = PBF_KEGG, aes(x = Description, y = RichFactor,
                            size = Count,
                            color = qvalue)) +
  geom_point(aes(reorder(Description, -qvalue), y=RichFactor)) +
  coord_flip() +
  theme_linedraw(base_size = 12) +
  labs(x = "Pathway", size="GeneNumber", color="qValue ") +
  scale_color_gradient(low="red", high="blue") +
  scale_size_continuous(limits = c(1, 4),
                        range = c(1,8)) +
  facet_wrap(~Regulation) +
  scale_y_continuous(labels = label_number(accuracy = 0.01)) +
  theme(axis.text.x = element_text(angle = 45,
                                   hjust = 1))

###################################################### PS

# PS_down_geneList <- read.csv("KEGG/KEGG_PS_down.csv", header = TRUE, row.names = 1)
# PS_down_geneList <- row.names(PS_down_geneList)
# PS_down_geneList
# 
# PS_down <- enrichKEGG(gene = PS_down_geneList,
#                       organism = "pvm",
#                       keyType = "ncbi-geneid",
#                       universe = unique(vannamei_db$Geneid),
#                       pAdjustMethod = "BH",
#                       use_internal_data = FALSE,
#                       pvalueCutoff = 1)
# 
# #write.csv(PS_down, "KEGG/PS_DOWN_KEGG.csv")
# 
# dotplot(PS_down, showCategory = 10)
# 
# PS_up_geneList <- read.csv("KEGG/KEGG_PS_up.csv", header = TRUE, row.names = 1)
# PS_up_geneList <- row.names(PS_up_geneList)
# PS_up_geneList
# 
# PS_up <- enrichKEGG(gene = PS_up_geneList,
#                     organism = "pvm",
#                     keyType = "ncbi-geneid",
#                     universe = unique(vannamei_db$Geneid),
#                     pAdjustMethod = "BH",
#                     use_internal_data = FALSE,
#                     pvalueCutoff = 1)
# 
# #write.csv(PS_up, "KEGG/PS_UP_KEGG.csv")
# 
# dotplot(PS_up, showCategory = 10)

PS_KEGG <- read.csv("KEGG/PS_KEGG.csv", header = TRUE)

PS_KEGG_plot <- ggplot(data = PS_KEGG, aes(x = Description, y = RichFactor,
                           size = Count,
                           color = qvalue)) +
  geom_point(aes(reorder(Description, -qvalue), y=RichFactor)) +
  coord_flip() +
  theme_linedraw(base_size = 12) +
  labs(x = "Pathway", size="GeneNumber", color="qValue        ") +
  scale_size_continuous() +
  scale_color_gradient(low="red", high="blue") +
  facet_wrap(~Regulation) +
  scale_y_continuous(labels = label_number(accuracy = 0.01)) +
  theme(axis.text.x = element_text(angle = 45,
                                   hjust = 1))

###################################################### NS
# 
# NS_down_geneList <- read.csv("KEGG/KEGG_NS_down.csv", header = TRUE, row.names = 1)
# NS_down_geneList <- row.names(NS_down_geneList)
# NS_down_geneList
# 
# NS_down <- enrichKEGG(gene = NS_down_geneList,
#                       organism = "pvm",
#                       keyType = "ncbi-geneid",
#                       universe = unique(vannamei_db$Geneid),
#                       pAdjustMethod = "BH",
#                       use_internal_data = FALSE,
#                       pvalueCutoff = 1)
# 
# #write.csv(NS_down, "KEGG/NS_DOWN_KEGG.csv")
# 
# dotplot(NS_down, showCategory = 10)
# 
# NS_up_geneList <- read.csv("KEGG/KEGG_NS_up.csv", header = TRUE, row.names = 1)
# NS_up_geneList <- row.names(NS_up_geneList)
# NS_up_geneList
# 
# NS_up <- enrichKEGG(gene = NS_up_geneList,
#                     organism = "pvm",
#                     keyType = "ncbi-geneid",
#                     universe = unique(vannamei_db$Geneid),
#                     pAdjustMethod = "BH",
#                     use_internal_data = FALSE,
#                     pvalueCutoff = 1)
# 
# #write.csv(NS_up, "KEGG/NS_UP_KEGG.csv")
# 
# dotplot(NS_up, showCategory = 10)

NS_KEGG <- read.csv("KEGG/NS_KEGG.csv", header = TRUE)

NS_KEGG <- NS_KEGG %>%
  arrange(qvalue)
# Force top 20
NS_KEGG <- head(NS_KEGG,20) 

NS_KEGG_plot <- ggplot(data = NS_KEGG, aes(x = Description, y = RichFactor,
                           size = Count,
                           color = qvalue)) +
  geom_point(aes(reorder(Description, -qvalue), y=RichFactor)) +
  coord_flip() +
  theme_linedraw(base_size = 12) +
  labs(x = "Pathway", size="GeneNumber", color="qValue") +
  scale_color_gradient(low="red", high="blue") +
  scale_size_continuous() +
  facet_wrap(~Regulation) +
  scale_y_continuous(labels = label_number(accuracy = 0.01)) +
  theme(axis.text.x = element_text(angle = 45,
                                   hjust = 1))

######################################################
######################################################
######################################################
######################################################
######################################################
######################################################
##################### Figure Work ####################
######################################################
######################################################
######################################################
######################################################
######################################################
######################################################

CF_CF_volcano_plot
CF_CF_heatmap <- as.ggplot(CF_CF_heatmap)
CF_CF_PCA_plot
CF_CF_distance_matrix
CF_UP_GO
CF_DOWN_GO
CF_KEGG_plot

PBF_PBF_volcano_plot
PBF_PBF_heatmap <- as.ggplot(PBF_PBF_heatmap)
PBF_PBF_PCA_plot
PBF_PBF_distance_matrix
PBF_UP_GO
PBF_DOWN_GO
PBF_KEGG_plot

PS_volcano_plot
PS_heatmap <- as.ggplot(PS_heatmap)
PS_PCA_plot
PS_distance_matrix
PS_UP_GO
PS_DOWN_GO
PS_KEGG_plot

NS_volcano_plot
NS_heatmap <- as.ggplot(NS_heatmap)
NS_PCA_plot
NS_distance_matrix
NS_UP_GO
NS_DOWN_GO
NS_KEGG_plot

CF_1 <- ggarrange(CF_CF_volcano_plot,
          CF_CF_heatmap,
          CF_CF_PCA_plot,
          labels = c("A", "B", "C"),
          hjust= 0,
          vjust = 1,
          nrow = 1,
          ncol = 3,
          font.label = list(size = 18, 
                            face = "bold", 
                            color ="black"),
          align = "none",
          common.legend = TRUE,
          legend = "none")

CF_2 <- ggarrange(CF_UP_GO + rremove("x.title"),
                  CF_DOWN_GO + rremove("x.title"),
                  CF_KEGG_plot,
                  labels = c("D", "E", "F"),
                  hjust= 0,
                  vjust = 1,
                  nrow = 3,
                  ncol = 1,
                  font.label = list(size = 18,
                                    face = "bold",
                                    color ="black"),
                  align = "h",
                  common.legend = FALSE,
                  legend = "right")

ggarrange(CF_1,
          CF_2,
          ncol = 1,
          heights = c(0.75,2))

PBF_1 <- ggarrange(PBF_PBF_volcano_plot,
                  PBF_PBF_heatmap,
                  PBF_PBF_PCA_plot,
                  labels = c("A", "B", "C"),
                  hjust= 0,
                  vjust = 1,
                  nrow = 1,
                  ncol = 3,
                  font.label = list(size = 18, 
                                    face = "bold", 
                                    color ="black"),
                  align = "none",
                  common.legend = TRUE,
                  legend = "none")

PBF_2 <- ggarrange(PBF_UP_GO + rremove("x.title"),
                  PBF_DOWN_GO + rremove("x.title"),
                  PBF_KEGG_plot,
                  labels = c("D", "E", "F"),
                  hjust= 0,
                  vjust = 1,
                  nrow = 3,
                  ncol = 1,
                  font.label = list(size = 18,
                                    face = "bold",
                                    color ="black"),
                  align = "h",
                  common.legend = FALSE,
                  legend = "right")

ggarrange(PBF_1,
          PBF_2,
          ncol = 1,
          heights = c(0.75,2))

PS_1 <- ggarrange(PS_volcano_plot,
                  PS_heatmap,
                  PS_PCA_plot,
                  labels = c("A", "B", "C"),
                  hjust= 0,
                  vjust = 1,
                  nrow = 1,
                  ncol = 3,
                  font.label = list(size = 18, 
                                    face = "bold", 
                                    color ="black"),
                  align = "none",
                  common.legend = TRUE,
                  legend = "none")

PS_2 <- ggarrange(PS_UP_GO + rremove("x.title"),
                  PS_DOWN_GO + rremove("x.title"),
                  PS_KEGG_plot,
                  labels = c("D", "E", "F"),
                  hjust= 0,
                  vjust = 1,
                  nrow = 3,
                  ncol = 1,
                  font.label = list(size = 18,
                                    face = "bold",
                                    color ="black"),
                  align = "h",
                  common.legend = FALSE,
                  legend = "right")

ggarrange(PS_1,
          PS_2,
          ncol = 1,
          heights = c(0.75,2))

NS_1 <- ggarrange(NS_volcano_plot,
                  NS_heatmap,
                  NS_PCA_plot,
                  labels = c("A", "B", "C"),
                  hjust= 0,
                  vjust = 1,
                  nrow = 1,
                  ncol = 3,
                  font.label = list(size = 18, 
                                    face = "bold", 
                                    color ="black"),
                  align = "none",
                  common.legend = TRUE,
                  legend = "none")

NS_2 <- ggarrange(NS_UP_GO + rremove("x.title"),
                  NS_DOWN_GO + rremove("x.title"),
                  NS_KEGG_plot,
                  labels = c("D", "E", "F"),
                  hjust= 0,
                  vjust = 1,
                  nrow = 3,
                  ncol = 1,
                  font.label = list(size = 18,
                                    face = "bold",
                                    color ="black"),
                  align = "h",
                  common.legend = FALSE,
                  legend = "right")

ggarrange(NS_1,
          NS_2,
          ncol = 1,
          heights = c(0.75,2))

