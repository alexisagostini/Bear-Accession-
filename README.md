# Start the bear project
## the analyse 

```bash
conda install -y -c bioconda -c conda-forge bcftools vcftools plink plink2
Final_variant = /home/alexis/data/Ours/results/final_variants.clean.vcf.gz
Repo = /home/alexis/data/Ours/results/
cd Repo


# New filter MAT 
vcftools --gzvcf results_numenius/final_variants.clean.max_missing_0.9.recode.vcf.gz   --maf 0.05   --recode --stdout | bgzip -c > results_numenius/output.maf05.vcf.gz

# Index it
 tabix -p vcf results_numenius/output.maf05.vcf.gz

# Pipeline plink

plink --vcf output.maf05.vcf.gz --allow-extra-chr --set-missing-var-ids @:# --make-bed --out data

plink --bfile data --allow-extra-chr --indep-pairwise 50 5 0.2 --out prune

plink --bfile data --allow-extra-chr --extract prune.prune.in --make-bed --out data_pruned

# Output PCA

plink --bfile data_pruned --allow-extra-chr --pca --out PCA
```
## the plots
```R
library(ggplot2)

pca <- read.table("PCA.eigenvec.txt", header=FALSE)

colnames(pca)[1:2] <- c("FID", "IID")

colnames(pca)[3:ncol(pca)] <- paste0("PC", 1:(ncol(pca)-2))

ggplot(pca, aes(x = PC1, y = PC2)) +
  geom_point(size = 3) +
  theme_minimal() +
  xlab("PC1") +
  ylab("PC2") +
  ggtitle("PCA - Numenius arquata")


meta <- read.table("meta_2.txt", header = TRUE)

pca <- merge(pca, meta, by = "IID")

ggplot(pca, aes(x = PC1, y = PC2, color = Pop)) +
  geom_point(size = 3, alpha = 0.8) +
  theme_minimal() +
  labs(
    title = "PCA - Numenius arquata and Numenius phaeopus",
    x = "PC1",
    y = "PC2",
    color = "Population"
  )

ggplot(pca, aes(x = PC1, y = PC2, color = Pop)) +
  geom_point(size = 3, alpha = 0.8) +
  theme_minimal() +
  labs(
    title = "PCA - Numenius arquata",
    x = "PC1",
    y = "PC2",
    color = "Population"
  )

ggplot(pca, aes(PC1, PC2, color = Pop)) +
  geom_point(size = 3) +
  stat_ellipse() +
  theme_minimal()

  # Calcul de ROH
  plink --bfile data_pruned \
--allow-extra-chr \
--homozyg \
--out ROH
```
Script based on Raquel Mejia analysis https://docs.google.com/presentation/d/17WGN-y_bwqKkUH9UaBf3J2hHKcvSfTc_IgLPYdx91yg/edit?slide=id.g3e0c63b595d_0_5#slide=id.g3e0c63b595d_0_5*

## second analysis 
