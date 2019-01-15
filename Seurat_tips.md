

### Merge a lot of Seurat objects

load libraries 

```r
#install.packages("tidyverse")
#install.packages("Seurat")
# here package is to get rid of absolute path
# see details https://github.com/jennybc/here_here
#install.packages("here")
# interact with file systems https://github.com/r-lib/fs
#install.packages("fs")
library(tidyverse)
library(Seurat)
library(here)
library(fs)
library(purrr)
here()
```

read in the data as seruat object
and then merge https://github.com/satijalab/seurat/issues/480

```{r}
input_folders<- dir_ls( path = here("data"), recursive = T) %>% path_dir() %>%
        unique() %>% str_subset(""mm10-1.2.0_premrna")

samples<- str_extract(input_folders, pattern = yourpattern)
names(input_folders)<- samples
seurat_data<- purrr::map(input_folders, Read10X)

#prefix the sample name to the cell name, otherwise merge seurat objects gives error
add_sample_name_to_cell<- function(x, y){
        colnames(x)<- paste(y, colnames(x), sep = "_")
        return(x)
}

seurat_data<- map2(seurat_data, samples, add_sample_name_to_cell)
seurat_objects<- map2(seurat_data, samples, function(x,y) CreateSeuratObject(raw.data = x, project = y))
```

merge

```{r}

merged_seurat<- purrr::reduce(seurat_objects, function(x,y) {MergeSeurat(x,y, do.normalize = F)})
```

### SubsetData

https://github.com/satijalab/seurat/issues/563


>When we run SubsetData, we have (by default) not subsetted the raw.data slot as well, as this can be slow and 
usually unnecessary. This can in some cases cause problems downstream, but setting do.clean=T does a full subset.


### RunPCA

https://satijalab.org/seurat/dim_reduction_vignette.html


>In Seurat v2.0, we have switched all PCA calculations to be performed via the irlba package to enable calculation of partial PCAs (i.e. only calculate the first X PCs). While this is an approximate algorithm, it performs remarkably similar to running a full PCA and has significant savings in terms of computation time and resources. These savings become necessary when running Seurat on increasingly large datasets. We also allow the user to decide whether to weight the PCs by the percent of the variance they explain (the weight.by.var parameter). **For large datasets containing rare cell types, we often see improved results by setting this to FALSE, as this prevents the initial PCs (which often explain a disproportionate amount of variance)** 

### log transformation

**logNormalize in seurat is natural log** https://github.com/satijalab/seurat/issues/171 and `avg_diff` in the `FindMarkers` output is also natural log https://github.com/satijalab/seurat/issues/77


### some notes from Sophia Liang in Dulac lab


* I run Jackstraw to determine how many PCs to use, instead of looking at each PCHeatmaps. I found it gives better cluster results and now use it before each clustering step.

*  In Seurat, the default neighboring number to group when running a tSNE is 30. may need to test different param.k.
### choosing number of PCs for finding clusters

see an issue I opened https://github.com/satijalab/seurat/issues/1058

>Choosing the dimensionality is a fundamental problem in single cell analysis and I don't think there is a fully unsupervised or principled way to choose the number of PCs to use. In general I look at the PC elbow plot, use more components for large or more heterogeneous datasets, and when in doubt I include more rather than fewer components. However, I would recommend using the same dimensions for clustering and UMAP or tSNE (ie, if you use 1:9 for clustering you should also use 1:9 for visualization).

