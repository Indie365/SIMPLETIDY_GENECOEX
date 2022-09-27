# Simple Tidy GeneCoEx 
A simple gene co-expression analyses workflow powered by tidyverse and graph analyses


# Introduction 
This is a gene co-expression analysis workflow powered by tidyverse and graph analyses. 
The essence of this workflow is simple and tidy. 
This is by no means the best workflow, but it is conceptually simple if you are familiar with tidyverse. 
The goal of this workflow is identify genes co-expressed with known genes of interest. 

* Author: Chenxin Li, Postdoctoral Research Associate, Center for Applied Genetic Technologies, University of Georgia
* Contact: Chenxin.Li@uga.edu 

## Example data 
We will be using the [Shinozaki et al., 2018](https://www.nature.com/articles/s41467-017-02782-9 ) tomato fruit developmental transcriptomes as our practice data.
This dataset contains 10 developmental stages and 11 tissues. 
The goal of this example is to identify genes co-expressed with known players of fruit ripening. 
The expression matrix is available [online](https://doi.org/10.5281/zenodo.7117357) as a .gz file. 
You can gunzip it and move it into the `Data/` directory. 

# Dependencies 
```{r}
library(tidyverse)
library(igraph)
library(ggraph)

library(readxl)
library(patchwork)
library(RColorBrewer)
library(viridis)

set.seed(666)
```
The [tidyverse](https://www.tidyverse.org/) and [igraph](https://igraph.org/) packages will be doing a lot of the heavy lifting. 
[ggraph](https://ggraph.data-imaginist.com/) is a grammar of graphics extension for `igraph`, which provides effective visualization of network graphs. 

The rest of the packages are mainly for data visualization and not required for the gene expression analyses. 
The package `readxl` is only required if you have any files in `.xlsx` or `.xlx` format (anything only Excel readable). 

The `Scripts/` directory contains `.Rmd` files that generate the graphics shown below. 
It requires R, RStudio, and the rmarkdown package. 

* R: [R Download](https://cran.r-project.org/bin/)
* RStudio: [RStudio Download](https://www.rstudio.com/products/rstudio/download/)
* rmarkdown can be installed using the intall packages interface in RStudio


# Required input
The workflow requires 3 input. 

1. Gene expression matrix 
2. Metadata 
3. Bait genes (genes involved in the biological process of interest from previous studies) 

## Gene expression matrix
Many software can generate gene expression matrix, such as [Cufflinks](http://cole-trapnell-lab.github.io/cufflinks/), [kallisto](https://pachterlab.github.io/kallisto/about), and [STAR](https://github.com/alexdobin/STAR). 

My go-to is kallisto, but you do you. The requirements are:

* Estimation of gene expression abundance, in units of TPM or FPKM. 
* Each row is a gene, and each column is a library. 

```{r}
Exp_table <- read_csv("../Data/Shinozaki_tpm_all.csv", col_types = cols())
head(Exp_table)
dim(Exp_table)
```

```
# [1] 66880 484
```
Looks like there are 66880 genes and 484 columns. Since the 1st column is gene IDs, there are total of 483 libraries.

## Metadata
Metadata are *very* helpful for any gene expression analyses. 
Metadata are the data of the data, the biological and technical descriptions for each library. 

* If you downloaded your data from [SRA](https://www.ncbi.nlm.nih.gov/sra), you can fetch the metadata associated with the submission. You can use [E-utilities](https://www.ncbi.nlm.nih.gov/books/NBK179288/) to fetch metadata given an accession number. 
* If you are analyzing unpublished data, contact your colleagues who generated the samples for metadata.

```{r}
Metadata <- read_excel("../Data/Shinozaki_datasets_SRA_info.xlsx")
head(Metadata)
dim(Metadata)
```

```
# [1] 483  17
```
Looks like there are 483 libraries and 17 different technical or biological descriptions for each library. 
**At this step, you should check that the number of libraries matches between the metadata and gene expression matrix.**
In this case, both indicate there are 483 libraries, so we are good to proceed. 

## Bait genes 
It is rare to go into a transcriptome completely blind (not knowing anything about the biology). Not impossible, but rare. 
Oftentimes, we are aware of some "bait genes", genes that are previously implicated in the biological processes in question.

In this example, we have two bait genes, `PG` and `PSY1`. 

* `PG` is involved in making the fruit soft [review](https://www.annualreviews.org/doi/pdf/10.1146/annurev.pp.42.060191.003331).
* `PSY1` is involved in producing the red color of the fruit [ref](https://link.springer.com/article/10.1007/BF00047400). 

```{r}
Baits <- read_delim("../Data/Genes_of_interest.txt", delim = "\t", col_names = F, col_types = cols())
head(Baits)
```
For the purpose of this example, we will just use two bait genes. 
The gene IDs for these two genes are also recorded in this small table. 
For an actual study, the bait gene list could be very long. 
You would probably include functional annotations and references as columns of the bait gene table.

# Understanding the experimental design
Before I start doing any analyses I would first try to wrap my head around the experimental design. 
Having a good understanding of the experimental design helps me decide how I want to analyze and visualize the data. 

Key questions are:

* What are the sources of variation?
* What are the levels of replication?

This is where the metadata come in handy.
## Major factors in the experiment

```{r}
Metadata %>% 
  group_by(dev_stage) %>% 
  count()
```

```
# A tibble:16 × 2 Groups:dev_stage [16]
```
According to the metadata, there are 16 developmental stages. 
According to the paper, the order of the developmental statges are:

1. Anthesis
2. 5 DAP
3. 10 DAP
4. 20 DAP
5. 30 DAP
6. MG
7. Br
8. Pk
9. LR
10. RR

Now this is a problem. The paper indicates less developmental stages than the metadata. How? 
Inspecting the metadata, each of MG, Br, and PK are subdivided into 3 "stages" - stem, equatorial, and stylar. 
But these "stages" are not time points, they are refering to location of the fruit. 
We will have to fix this later. 

```{r}
Metadata %>% 
  group_by(tissue) %>% 
  count()
```

```
# A tibble:11 × 2 Groups:tissue [11]
```
Looks like there are 11 tissues. The paper also indicates there are 11 tissues. We are good here. 

## Levels of replication
```{r}
Metadata %>% 
  group_by(tissue, dev_stage) %>% 
  count()
```

```
# A tibble:133 × 3 Groups:tissue, dev_stage [133]
```
Looks like there are 133 tissue * "developmental stage" combination. 
Some have 3 reps; some have 4. That's ok. 

## Summary of experimental design
This is a two factor experimental design: developmental stage * tissue. 
The major sources of variations are developmental stages, tissues, and replicates. 
I usually make a summary table to guide my downstream analyses. 

| source | type     | levels   | 
|:------:|:--------:|:--------:|
| Tissue | Qual     | 11       |
| Dev.   | Num/qual | 16 or 10 |
| Reps   | EU, OU   | 483      | 


The source column indicates the sources of variations. This will become important when we try to understand the major driver of variance in this experiment. 
The type column indicates if the factor in question is a qualitative (discrete) or numeric variable. 
A special note is that developmental stages can be either analyzed as numeric variable or a qualitative variable.
"EU" and "OU" in the Reps row stands for experimental unit and observational unit. In this case, the rep is both EU and OU. 
This is not always the case, especially if the same library is sequenced twice and uploaded with two different SRA number. 

# Global view of the experiment 
Now we understand the experimental design, we will figure out what is the major driver of variance in the experiment next.
In other words, between developmental stage and tissue, which factor contributes more to the variance in this experiment? 
The answer to this question matters in terms of how we mostly effectively visualize our data. 

A good way to have a global view of the experiment is doing a principal component analysis (PCA).
*This is a tidyverse workflow, so I will be doing things in the tidyverse way.* Brace yourself for `%>%`.

The first thing for tidyverse workflow is going to from wide format to tidy (or long format).
In tidy format, each row is an observation, and each column is a variable.
We can go from wide to long using the `pivot_longer()` function. 

```{r}
Exp_table_long <- Exp_table %>% 
  rename(gene_ID = `...1`) %>% 
  pivot_longer(cols = !gene_ID, names_to = "library", values_to = "tpm") %>% 
  mutate(logTPM = log10(tpm + 1)) 

head(Exp_table_long)
```
In this code chunk, I also renamed the first column to "gene_ID" and log transformed the tpm values. 
All in one pipe. We will come back to this long table later. This long table is the basis of all downstream analyses. 

## PCA 
However, the input data for PCA is a numeric matrix, so we have to go from long to wide back again. 
To do that, we use `pivot_wider()`.

```{r}
Exp_table_log_wide <- Exp_table_long %>% 
  select(gene_ID, library, logTPM) %>% 
  pivot_wider(names_from = library, values_from = logTPM)

head(Exp_table_log_wide)
```

```{r}
my_pca <- prcomp(t(Exp_table_log_wide[, -1]))
pc_importance <- as.data.frame(t(summary(my_pca)$importance))
head(pc_importance, 20)
```

```
# $1 Standard deviation
# $2 Proportion of Variance
# $3 Cumulative Proportion

# PC1	55.856580	0.43662	0.43662	
# PC2	27.601642	0.10662	0.54323	
# PC3	18.916665	0.05008	0.59331	
# PC4	15.105094	0.03193	0.62524	
# PC5	13.465655	0.02538	0.65062	
# PC6	11.751300	0.01933	0.66994	
# PC7	9.454201	0.01251	0.68245	
# PC8	8.560489	0.01026	0.69271	
# PC9	8.193150	0.00939	0.70210	
# PC10	8.105687	0.00919	0.71129
```
`prcomp()` performs PCA for you, given a numeric matrix, which is just the transposed `Exp_table_log_wide`, but without the gene ID column. 
`as.data.frame(t(summary(my_pca)$importance))` saves the sd and proportion of variance into a data table. 
In this case, the 1st PC accounts for 43% of the variance in this experiment.
The 2nd PC accounts for 10% of the variance.  

## Graph PCA plot 
To make a PCA plot, we will graph the data stored in `my_pca$x`, which stores the coordinates of each library in PC space. 
Let's pull that data out and annotate them (with metadata). 

```{r}
PCA_coord <- my_pca$x[, 1:10] %>% 
  as.data.frame() %>% 
  mutate(Run = row.names(.)) %>% 
  full_join(Metadata %>% 
              select(Run, tissue, dev_stage, `Library Name`, `Sample Name`), by = "Run")

head(PCA_coord)
```
For the purpose of visualization, I only pulled the first 10 PC. In fact, I will be only plotting the first 2 or 3 PCs. 
For the purpose of analysis, I only pulled the biologically relevant columns from the metadata: Run, tissue, dev_stage, Library Name, and Sample Name. 

We noticed that there were in fact only 10 developmental stages, so let's fix that here. 

```{r}
PCA_coord <- PCA_coord %>% 
  mutate(stage = case_when(
    str_detect(dev_stage, "MG|Br|Pk") ~ str_sub(dev_stage, start = 1, end = 2),
    T ~ dev_stage
  )) %>% 
  mutate(stage = factor(stage, levels = c(
   "Anthesis",
   "5 DPA",
   "10 DPA",
   "20 DPA",
   "30 DPA",
   "MG",
   "Br",
   "Pk",
   "LR",
   "RR"
  ))) %>% 
  mutate(dissection_method = case_when(
    str_detect(tissue, "epidermis") ~ "LM",
    str_detect(tissue, "Collenchyma") ~ "LM",
    str_detect(tissue, "Parenchyma") ~ "LM",
    str_detect(tissue, "Vascular") ~ "LM",
    str_detect(dev_stage, "Anthesis") ~ "LM",
    str_detect(dev_stage, "5 DPA") &
      str_detect(tissue, "Locular tissue|Placenta|Seeds") ~ "LM",
    T ~ "Hand"
  ))

head(PCA_coord)
```
I made a new `stage` column, and parse the old `dev_stage` column. If `dev_stage` were MG, Br, or Pk, only keep the first two characters. 
I also manually reordered the stages. It's good to have biological meaningful orders. 
I could have also ordered the tissue column in some way, e.g., from outer layer of the fruit to inner layer. We can do that if it turns out to be necessary. 

According to the paper, 5 pericarp tissues were collected using laser capture microdissection (LM), so I parsed those out: 

* Outer and inner epidermis
* Collenchyma
* Parenchyma
* Vascular tissue 

In addition, some early stage samples were also collected uisng LM:

> Due to their small size, laser  microdissection (LM) was used to harvest these six tissues at anthesis, as well as locular tissue, placenta, and seeds at 5 DPA.

```{r}
PCA_by_method <- PCA_coord %>% 
  ggplot(aes(x = PC1, y = PC2)) +
  geom_point(aes(fill = dissection_method), color = "grey20", shape = 21, size = 3, alpha = 0.8) +
  scale_fill_manual(values = brewer.pal(n = 3, "Accent")) +
  labs(x = paste("PC1 (", pc_importance[1, 2] %>% signif(3)*100, "% of Variance)", sep = ""), 
       y = paste("PC2 (", pc_importance[2, 2] %>% signif(3)*100, "% of Variance)", "  ", sep = ""),
       fill = NULL) +  
  theme_bw() +
  theme(
    text = element_text(size= 14),
    axis.text = element_text(color = "black")
  )

PCA_by_method

ggsave("../Results/PCA_by_dissection_method.svg", height = 3, width = 4, bg = "white")
ggsave("../Results/PCA_by_dissection_method.png", height = 3, width = 4, bg = "white")
```
![PCA_by_dissection_method.svg](https://github.com/cxli233/SimpleTidy_GeneCoEx/blob/main/Results/PCA_by_dissection_method.svg)

First thing to watch out for is technical differences. It seems the dissection method IS the major source of variance, corresponding perfectly to PC1. 

For biological interpretation, it's then better to look at PC2 and PC3.
```{r}
PCA_by_tissue <- PCA_coord %>% 
  ggplot(aes(x = PC2, y = PC3)) +
  geom_point(aes(fill = tissue), color = "grey20", shape = 21, size = 3, alpha = 0.8) +
  scale_fill_manual(values = brewer.pal(11, "Set3")) +
  labs(x = paste("PC2 (", pc_importance[2, 2] %>% signif(3)*100, "% of Variance)", sep = ""), 
       y = paste("PC3 (", pc_importance[3, 2] %>% signif(3)*100, "% of Variance)", "  ", sep = ""),
       fill = "tissue") +  
  theme_bw() +
  theme(
    text = element_text(size= 14),
    axis.text = element_text(color = "black")
  )

PCA_by_stage <- PCA_coord %>% 
  ggplot(aes(x = PC2, y = PC3)) +
  geom_point(aes(fill = stage), color = "grey20", shape = 21, size = 3, alpha = 0.8) +
  scale_fill_manual(values = viridis(10, option = "D")) +
  labs(x = paste("PC2 (", pc_importance[2, 2] %>% signif(3)*100, "% of Variance)", sep = ""), 
       y = paste("PC3 (", pc_importance[3, 2] %>% signif(3)*100, "% of Variance)", "  ", sep = ""),
       fill = "stage") +  
  theme_bw() +
  theme(
    text = element_text(size= 14),
    axis.text = element_text(color = "black")
  )

wrap_plots(PCA_by_stage, PCA_by_tissue, nrow = 1)
ggsave("../Results/PCA_by_stage_tissue.svg", height = 3.5, width = 8.5, bg = "white")
ggsave("../Results/PCA_by_stage_tissue.png", height = 3.5, width = 8.5, bg = "white")
```
![PCA_by_stage_tissue.svg](https://github.com/cxli233/SimpleTidy_GeneCoEx/blob/main/Results/PCA_by_stage_tissue.svg)

Now the x-axis (PC2) clearly separates developmental stages young to old from left to right. 
The y-axis (PC3) clearly separates seeds from everything else. 

Thus, in terms of variance contribution, dissection method > stage > tissue. 
We will use this information to guide downstream visualization. 

Now we have to make a decision. 
The fact that the major driver of variation is a technical factor may be a concern. 
Perhaps LM samples are lower input and thus lower library complexity? I don't know.
But to best separate biological variation from technical variation, we should do separate gene co-expression analyses for hand collected and LM samples. 

For the sake of this exercise, let's focus on hand collected samples. 

# Gene co-expression analyses 
All of the above are preparatory work. It helps us understand the data.
Now we are ready to do co-expression analyses. 

There are multiple steps. Let's go over them one by one. 

## Average up the reps 
We will first average up the reps to the level of tissue-stage combination. 
We are interested in the biological variation among tissue-stage combination, and less interested in the noise among reps of the same treatment. 
Again, this is a *tidyverse based workflow*. 

```{r}
Exp_table_long_averaged <- Exp_table_long %>% 
  full_join(PCA_coord %>% 
              select(Run, `Sample Name`, tissue, dev_stage, dissection_method), 
            by = c("library"="Run")) %>% 
  filter(dissection_method == "Hand") %>% 
  group_by(gene_ID, `Sample Name`, tissue, dev_stage) %>% 
  summarise(mean.logTPM = mean(logTPM)) %>% 
  ungroup()  

head(Exp_table_long_averaged)
```

We start from the long (tidy) table we made earlier. I also pulled the metadata as well to guide the averaging process. 
`by = c("library"="Run)` inside `full_join()` deals with the fact that the library ID is called `library` in the long table, but `Run` in the metadata. 
Then we filter for `dissection_method == "Hand`. 
`group_by()` followed by `summarise(mean = ...)` takes each gene, tissue, and dev_stage, and computes the mean. 
The elegance of a tidyverse based workflow is that you do not have to do loops! You let `group_by()` do the heavy lifting. 
This could take a moment. This step is doing a lot of mean calculations. 

## Z score
Once we averaged up the reps, we will standardize the expression pattern using z score. 
A z score is the difference from mean over the standard deviation.
It standardize the expression pattern of each gene to mean = 0, sd = 1. 
It is not absolutely necessary, but I have found including this step to produce results that better capture the underlying biology.

```{r}
Exp_table_long_averaged_z <- Exp_table_long_averaged %>% 
  group_by(gene_ID) %>% 
  mutate(z.score = (mean.logTPM - mean(mean.logTPM))/sd(mean.logTPM)) %>% 
  ungroup()

head(Exp_table_long_averaged_z)
```
In this step, we are grouping by gene. Tissue-stages with higher expression will have a higher z score and vice versa. 
Note that this is completely relative to each gene itself. 
Again, the advantage of a tidyverse workflow is you let `group_by()` do all the heavy lifting. No need for loops or `apply()`. 

## Gene selection
The next step is correlating each gene to every other gene. 
However, we have almost 67k genes in this dataset. The number of correlations scales to the square of number of genes. 
To make things faster and less cumbersome, we can select only the high variance genes. 
The underlying rationale is if a gene is expressed at a similar level across all samples, it is unlikely that is involved in the biology in a particular stage or tissue. 

There are multiple ways to selecting for high variance genes, and multiple cutoffs.
For example, you can calculate the gene-wise variance for all genes, and take the upper third. 
You can only take genes with a certain expression level (say > 5 tpm across all tissues), then take high variance gene. 
These are arbitrary. You do you. 

```{r}
high_var_genes <- Exp_table_long_averaged_z %>% 
  group_by(gene_ID) %>% 
  summarise(var = var(mean.logTPM)) %>% 
  ungroup() %>% 
  filter(var > quantile(var, 0.667))

head(high_var_genes)
dim(high_var_genes)
```

```
# [1] 22271     2
```

This chunk of code computes the variance for each gene. 
Again, this is completely relative to each gene itself. 
Then I filtered for top 33% high var genes. 

The above chunk just listed the high var genes, now we need to filter those out in the long table that contains the z-scores. 

For the sake of this example, let's just take top 5000 genes with highest var as a quick exercise.
You might want to take more genes in the analyses, but the more genes in the correlation, the slower everything will be.

```{r}
high_var_genes5000 <- high_var_genes %>% 
  slice_max(order_by = var, n = 5000) 

head(high_var_genes5000)
```
A good way to check if you have included enough genes in your analyses is to check if your bait genes are among the top var genes. 

```{r}
high_var_genes5000 %>% 
  filter(str_detect(gene_ID, Baits$X2[1]))

high_var_genes5000 %>% 
  filter(str_detect(gene_ID, Baits$X2[2]))
```

```
# A tibble:1 × 2
# A tibble:6 × 2
```

Both are present in the top 5000, so that's good. 
Note that incidentally, this gene expression matrix is at the level of isoforms. 
I would do it on only the representative gene models (longest gene model), but this particular matrix that I have access to is quantifying at the level of isoforms.

```{r}
Exp_table_long_averaged_z_high_var <- Exp_table_long_averaged_z %>% 
  filter(gene_ID %in% high_var_genes5000$gene_ID)

head(Exp_table_long_averaged_z_high_var)

Exp_table_long_averaged_z_high_var %>% 
  group_by(gene_ID) %>% 
  count() %>% 
  nrow()
```

```
# [1] 5000
```

The `%in%` operator filters gene_IDs that are present in `high_var_genes5000$gene_ID`, thus retaining only high var genes. 

## Gene-wise correlation
Now we can correlate each gene to every other gene. 
The essence of this workflow is simple, so we will use a simple correlation. 
If you want, you can use fancier methods such as [GENIE3](https://www.bioconductor.org/packages/devel/bioc/vignettes/GENIE3/inst/doc/GENIE3.html ) 

We will use the `cor()` function in R. But the `cor()` only take vector or matrix as input, so we need to go from long to wide again. 

```{r}
z_score_wide <- Exp_table_long_averaged_z_high_var %>% 
  select(gene_ID, `Sample Name`, z.score) %>% 
  pivot_wider(names_from = `Sample Name`, values_from = z.score) %>% 
  as.data.frame()

row.names(z_score_wide) <- z_score_wide$gene_ID
head(z_score_wide)
```
The `Sample Name` column contains info for both stage and tissue, which we can recall using the metadata. 
After long to wide transformation, the `Sample Name` column now becomes the column name of this wide table. 
Then we produce the correlation matrix. The underlying math here is R takes each column of a matrix and correlates it to every other columns. 
To get this to work on our wide table, we remove the `gene_ID` column, transpose it, and feed it into `cor()`.  

```{r}
cor_matrix <- cor(t(z_score_wide[, -1]))
dim(cor_matrix)
```

```
# [1] 5000 5000
```
This step can take a while, because it is computing many correlation coefficients. 
We threw in 5000 high var genes, so it is computing 5000^2 correlations. 
The correlation matrix should contain 5000 rows and 5000 columns. 

## Edge selection 
Now we have this huge correlation matrix, what do we do next? 
Not all correlation are statistical significant (whatever that means), and definitely not all correlation are biologically meaningful.
How do we select which correlations to use in downstream analyses. 
I call this step "edge selection", because this is building up to a network analysis, where each gene is node, and each correlation is an edge. 
I have two ways to do this. 

* t distribution approximation
* Empirical determination using rank distribution. 

### t distribution approximation. 
It turns out for each correlation coeff. r, you can approximate a t statistics, under some arbitrary assumptions. 
The equation is $$ t = r * \sqrt{(n-2) \over (1-r^2)} $$, where n is the number of observations. 
In this case, n is the number of tissue by stage combinations going into the correlation. Let's compute that first.

```{r}
number_of_tissue_stage <- ncol(z_score_wide) - 1
number_of_tissue_stage
```

```
# [1] 84
```

In this case, it is 84. There are two way to find it. 
The first way is the number of columns in the z score wide table - 1, because the 1st column is gene ID. 
The other way is using the parsed metadata, which is now part of `PCA_coord`. 

```{r}
PCA_coord %>% 
  filter(dissection_method == "Hand") %>% 
  group_by(tissue, dev_stage) %>% 
  count() %>% 
  nrow()
```

```
# [1] 84
```

Both methods say we have 84 unique tissue by stage combinations that were hand collected. 
We are good to proceed. 

```{r}
cor_matrix_upper_tri <- cor_matrix
cor_matrix_upper_tri[lower.tri(cor_matrix_upper_tri)] <- NA
```

Before we select edges (correlations), we need to deal with some redundant data. 
The correlation matrix is symmetrical along its diagonal. 
The diagonal will be 1, because it is correlating with itself.
Everything else appears twice. 
We can take care of that by setting the upper (or lower) triangle of this matrix to NA. 
This step can take a while. The larger the matrix, the slower it is. 

Now we can compute a t statistic from r and compute a p value using the t distribution. 
Again, this is a tidyverse workflow, so brace yourself for many `%>%`. 

```{r}
edge_table <- cor_matrix_upper_tri %>% 
  as.data.frame() %>% 
  mutate(from = row.names(cor_matrix)) %>% 
  pivot_longer(cols = !from, names_to = "to", values_to = "r") %>% 
  filter(is.na(r) == F) %>% 
  filter(from != to) %>% 
  mutate(t = r*sqrt((number_of_tissue_stage-2)/(1-r^2))) %>% 
  mutate(p.value = case_when(
    t > 0 ~ pt(t, df = number_of_tissue_stage-2, lower.tail = F),
    t <=0 ~ pt(t, df = number_of_tissue_stage-2, lower.tail = T)
  )) %>% 
  mutate(FDR = p.adjust(p.value, method = "fdr")) 

head(edge_table)
```

This chunk converts the correlation matrix into a data table. 
Then it goes from wide to long using `pivot_longer()`.
After that, everything is normal dyplr verbs, such as `mutate()` and `filter()`. 
P values are computed using the t distribution. 
Depending on the sign of t, the upper of lower tail probability is taken. 
Finally, the p values are adjusted for multiple comparisons using FDR. 
This step can take a while. Turning a large wide table to a long table always takes a while.
Your computer may not have enough memory to run this step if you put in many genes. 
In this case we only used 5000 genes, so no problem. 

You can look at various adjusted p value cutoffs and the corresponding r value before proceeding. 
Let's say we just look at positively correlated genes 

```{r}
edge_table %>% 
  filter(r > 0) %>% 
  filter(FDR < 0.05) %>% 
  slice_min(order_by = abs(r), n = 10)

edge_table %>% 
  filter(r > 0) %>% 
  filter(FDR < 0.01) %>% 
  slice_min(order_by = abs(r), n = 10)
```

```
# A tibble:10 × 6 
# from                  to                    r         t         p.value     FDR
# Solly.M82.02G011270.2	Solly.M82.03G001400.2	0.1958725	1.808737	0.03707863	0.04999967
# A tibble:10 × 6
# from                  to                    r         t         p.value     FDR
# Solly.M82.05G004690.1	Solly.M82.12G000310.1	0.2704730	2.544061	0.006416876	0.009999969
```

If you cut off the FDR at 0.05, then your r values are 0.196 or larger. 
If you cut off the FDR at 0.01, then your r values are 0.27  or larger. 
Not very high, but it is what it is. 

### Empirical determination using bait genes and rank distribution 
If I go into this analysis not knowing any biology, then I would proceed with a t approximation followed by some p value cutoff.
I think in real life, this is hardly the case. We usually know something a priori. 
This is where bait genes can be helpful. 
You can use the bait genes to determine the cutoff if you know two bait genes are involved in the same process. 
The underlying assumption is if two bait genes are involved in the same process, they might be co-expressed. 
Because this selection method is based on empirical observations, I argue this is better than using an arbitrary p value cutoff.

```{r}
edge_table %>% 
  filter(str_detect(from, "Solly.M82.03G005440") &
           str_detect(to,"Solly.M82.03G005440")) 
```

```
# A tibble:10 × 6 
# from                  to                    r         t         p.value       FDR
# Solly.M82.03G005440.1	Solly.M82.03G005440.2	0.9173865	20.87271	7.402139e-35	1.127654e-32
```
Different isoforms of the same gene is highly correlated, so that's good to see. 

```{r}
edge_table %>% 
  filter(str_detect(from, "Solly.M82.10G020850") &
           str_detect(to,"Solly.M82.03G005440") |
         str_detect(from, "Solly.M82.03G005440") &
           str_detect(to,"Solly.M82.10G020850")  ) 
```

```
# A tibble:6 × 6 
# from                  to                    r         t         p.value       FDR
# Solly.M82.03G005440.1	Solly.M82.10G020850.1	0.7872474	11.560813	3.356543e-19  4.772911e-18
```
These two bait genes (PG and PSY1) are chosen based on that they are involved in the same process.
They have a r value of from 0.73 to 0.78, which is rather high, considering at FDR < 0.01, r cutoff was 0.27. 

Base on this empirical observation, we can say we cut off at the vicinity of 0.73, maybe r > 0.7. 
Note that this is way more stringent than cutting off at FDR < 0.01 (r > 0.27). 

You can also look at the distribution of r values. 
```{r}
edge_table %>% 
  slice_sample(n = 20000) %>% 
  ggplot(aes(x = r)) +
  geom_histogram(color = "white", bins = 100) +
  geom_vline(xintercept = 0.7, color = "tomato1", size = 1.2) +
  theme_classic() +
  theme(
    text = element_text(size = 14),
    axis.text = element_text(color = "black")
  )

ggsave("../Results/r_histogram.svg", height = 3.5, width = 5, bg = "white")
ggsave("../Results/r_histogram.png", height = 3.5, width = 5, bg = "white")
```
![r_histogram.svg](https://github.com/cxli233/SimpleTidy_GeneCoEx/blob/main/Results/r_histogram.svg)
