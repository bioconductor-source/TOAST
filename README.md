# Analyses of high-throughput data from heterogeneous samples with TOAST

`TOAST` is an R package designed for the analyses of high-throughput data from complex, heterogeneous tissues. It is designed for the analyses of high-throughput data from 
  heterogeneous tissues,
  which is a mixture of different cell types.  
  
  TOAST offers functions for detecting cell-type 
  specific differential expression (csDE) or 
  differential methylation (csDM) for microarray data,
  and improving reference-free deconvolution 
  based on cross-cell type differential analysis. 
  TOAST implements a rigorous staitstical framework, 
  based on linear model, which provides great 
  flexibility for csDE/csDM detection and 
  superior computationl performance. 

## 1. Introduction

High-throughput technologies have revolutionized 
the genomics research.  The early
applications of the technologies were largely on 
cell lines. However, there is an increasing number 
of larger-scale, population level clinical studies 
in recent years, hoping to identify diagnostic 
biomarkers and therapeutic targets.  The samples 
collected in these studies, such as blood, tumor, 
or brain tissue, are mixtures of a number of different
cell types.  

The sample mixing complicates data analysis
because the experimental data from the high-throughput 
experiments are weighted average of signals from multiple
cell types. For these data, traditional analysis methods that ignores
the cell mixture 
will lead to results with low resolution, even biased or
errorneous results. 
For example, it has been discovered in epigenome-wide 
association studies (EWAS) that the mixing proportions 
can confound with the experimental factor of 
interest (such as age). Ignoring the cell mixing 
will lead to  false positives. 
On the other hand, cell type specific changes 
under different clinical conditions could be associated
with disease pathogenesis and progressions, and are of
great interests to researchers.

For heterogeneous samples, it is possible to profile the
cell types through experimental techniques.
They are, however, laborious and expensive that cannot
be applied to large scale studies.  
Computational tools for analzying the mixed data are
available for proportion estimation and cell type 
specific signal detection. 
<!-- deconvolution of signals and detection of signals.  -->
<!-- Without the experimental tools,  -->
There are a number of fundamental questions 
in the analyses: 
<!-- that are  for conducting in-silico 
deconvolution of signals and detection of signals.  -->

* First, how to estimate mixing proportions?  

     + There are a number of existing methods 
     that are devoted to solve this question.  
     These methods mainly can be categorized 
     to two groups: reference-based (require
     pure cell type profiles) and reference-free
     (does not require pure cell type profiles).  
     + It has been found that reference-based 
     deconvolution is more accurate and reliable
     than reference-free deconvolution.  
     + However, the reference panels required 
     for reference-based deconvolution can be 
     difficult to obtain.  
<!-- * As subsequence of the first question, we ask,
how to construct reference panels from available 
single-cell studies if reference-based deconvolution
is used?   -->
* If reference-free deconvolution is used, 
how to improve estimation accuracy? 
* Lastly, with available mixing proportions, 
how to detect cell-type specific DE/DM?

TOAST is a package designed to answer these 
questions and serve the research communities 
with tools for the analysis of heterogenuous 
tissues.  Currently TOAST provides functions 
to detect cell-type specific DE/DM, as well 
as differences across different cell types. 
TOAST also has functions to improve the 
accuracy of reference-free deconvolutions.

## 2. Installation and quick start

### Install TOAST


In R, install the `TOAST` package by

```{r install, message=FALSE, warning=FALSE}
library(devtools)
install_github("ziyili20/TOAST", build_vignettes=TRUE)
 
```

To view the package vignette in pdf format, run the following lines in R

```{r vig, message=FALSE, warning=FALSE}
library(TOAST)
vignette("TOAST")
```
The content in this README file is essentially the same as the package vignette.

### How to get help for TOAST

Any TOAST questions should be posted
to the GitHub Issue section of TOAST 
homepage at https://github.com/ziyili20/TOAST/issues.

### Quick start on detecting cell type-specific differential signals

Here we show the key steps for a cell 
type-specific different analysis. This 
code chunk assumes you have an expression
or DNA methylation matrix called `Y_raw`,
a data frame of sample information called
`design`, and a table of cellular composition
information (i.e. mixing proportions) 
called `prop`. If the cellular composition
is not available, the following sections 
will discuss about how to obtain mixing 
proportions using reference-free deconvolution 
or reference-based deconvolution.

```{r quick_start, eval = FALSE}
Design_out <- makeDesign(design, Prop)
fitted_model <- fitModel(Design_out, Y_raw)
fitted_model$all_coefs # list all phenotype names
fitted_model$all_cell_types # list all cell type names
# coef should be one of above listed phenotypes
# cell_type should be one of above listed cell types
res_table <- csTest(fitted_model, coef = "age", 
                    cell_type = "Neuron", contrast_matrix = NULL)
head(res_table)
```

## 3. Example datset

TOAST provides an example dataset of 450K 
DNA methylation. The following sections will
use this example dataset to demonstrate 
function usages. 

We obtain and process this dataset based 
on the raw data provided by GSE42861. This
is a DNA methylation 450K data for Rheumatoid
Arthiritis patients and controls. 
The original dataset has 485577 features
and 689 samples. We have reduced the dataset
to 3000 CpGs for randomly selected 50 RA patients
and 50 controls. 

```{r loaddata, loadData}
library(TOAST)
data("RA_100samples")
Y_raw <- RA_100samples$Y_raw
Pheno <- RA_100samples$Pheno
Blood_ref <- RA_100samples$Blood_ref
```

Check matrix including beta values for 
3000 CpG by 100 samples.

```{r checkData}
dim(Y_raw) 
Y_raw[1:4,1:4]
```

Check phenotype of these 100 samples.

```{r checkPheno}
dim(Pheno)
head(Pheno, 3)
```

Our example dataset also contain blood 
reference matrix for the matched 3000 
CpGs (obtained from bioconductor 
package `r Biocpkg("FlowSorted.Blood.450k")`.
```{r checkRef}
dim(Blood_ref)
head(Blood_ref, 3)
```


## 4. Estimate mixing proportions

If you have mixing proportions available, 
you can directly go to Section \@ref(section:csDE).

In many situations, mixing proportions 
are not readily available.  
There are a number of deconvolution methods
available to solve this problem.  To name a few:

* For DNA methylation: `r CRANpkg("RefFreeEWAS")`
(Houseman et al. 2016) is reference-free,  
and `r Biocpkg("EpiDISH")` (Teschendorff et al. 2017)
is reference-based.  

* For gene expression: qprog (Gong et al. 2011), 
deconf (Repsilber et al. 2010),  
lsfit (Abbas et al. 2009) 
and `r CRANpkg("DSA")` (Zhong et al. 2013).

In addition, [CellMix](https://github.com/rforge/cellmix)
package has summarized a number of deconvolution
methods and is a good resource to look up.  

Here we demonstrate two ways to estimate 
mixing proportions, one using 
`r CRANpkg("RefFreeEWAS")`, representing the
class of reference-free methods, and the other
using `r Biocpkg("EpiDISH")` (Teschendorff et al. 2017) as a 
representation of reference-based methods. 

We also provide function to improve reference-free
deconvolution performance in Section \@ref(section:ImpRF).
Note that in the following example we have only 
3000 features in the Y_raw dataset. Real 450K dataset should 
have around 485,000 features. More features generally
lead to more improvements, as feature selection 
become more important.



### 4.1 Reference-based deconvolution using least square method 

####1. Select the top 1000 most variant 
features by `findRefinx_var()`. 
To select the top features with 
largest coefficients of variations, 
one can use `findRefinx_cv()`.

```{r SelFeature}
refinx <- findRefinx_var(Y_raw, nmarker = 1000)
```

####2. Subset data and reference panel.

```{r Subset}
Y <- Y_raw[refinx,]
Ref <- as.matrix(Blood_ref[refinx,])
```

3. Use EpiDISH to solve cellular 
proportions and use post-hoc constraint.
```{r DB2}
library(EpiDISH)
outT <- epidish(beta.m = Y, ref.m = Ref, method = "RPC")
estProp_RB <- outT$estF
```

* **A word about Step 1** 

For step 1, one can also use `findRefinx_cv()` 
to select features based on coefficient of variantion. 
The choice of `findRefinx_var()` and `findRefinx_cv()`
depends on whether the feature variance of your data
highly correlated with feature mean. In gene expression
(microarray or RNA-seq) data, gene variance is highly
correlated with mean, thus `findRefinx_cv()` is recommended.
In DNA methylation data, this correlation is not 
strong, both `findRefinx_cv()` and `findRefinx_var()`
can be used. But we recommend `findRefinx_var()` 
for this situation since we find `findRefinx_var()`
has better feature selection for DNA methylation 
data than `findRefinx_cv()`(unpublished results).

```{r, DB3}
refinx = findRefinx_cv(Y_raw, nmarker=1000)
```


### 4.2 Reference-free deconvolution using RefFreeEWAS

####1. Load `r CRANpkg("RefFreeEWAS")`.

```{r DF, message=FALSE, warning=FALSE}
library(RefFreeEWAS)
```

####2. Similar to Reference-based deconvolution 
we also select the top 1000 most variant 
features by `findRefinx_var()`. And then subset data.

```{r DF2}
refinx <- findRefinx_var(Y_raw, nmarker = 1000)
Y <- Y_raw[refinx,]
```

####3. Do reference-free deconvolution on the RA dataset.

```{r, DF3, results='hide', message=FALSE, warning=FALSE}
K <- 6
outT <- RefFreeCellMix(Y, mu0=RefFreeCellMixInitialize(Y, K = K))
estProp_RF <- outT$Omega
```

####4. Comparing the reference-free method versus 
reference-base method

```{r compareRFRB}
# first we align the cell types from RF 
# and RB estimations using pearson's correlation
estProp_RF <- assignCellType(input=estProp_RF,
                             reference=estProp_RB) 
mean(diag(cor(estProp_RF, estProp_RB)))
```

### 4.3 Improve reference-free deconvolution with cross-cell type differential analysis

Feature selection is an important step 
before RF deconvolution and is directly
related with the estimation quality of 
cell composition.  `findRefinx.cv()` and
`findRefinx.var()` simply select the markers
with largest CV or largest variance, 
which may not always result in a good 
selection of markers.  Here, we propose
to improve RF deconvolution marker 
selection through cross-cell type 
differential analysis.  We implement
two versions of such improvement, 
one is for DNA methylation microarray 
data using `RefFreeCellMix` from `r CRANpkg("RefFreeEWAS")` 
package, the other one is for gene 
expression microarray data using `deconf`
from [CellMix](https://github.com/rforge/cellmix) package. 
To implement both, `r CRANpkg("RefFreeEWAS")` 
and [CellMix](https://github.com/rforge/cellmix) 
need to be installed first.

#### Improved-RF with RefFreeCellMix

#####1. Load TOAST package.

```{r IRB-RFCM1, message=FALSE, warning=FALSE}
library(TOAST)
```

#####2. Do reference-free deconvolution using 
improved-RF implemented with RefFreeCellMix. 
The default deconvolution function implemented
in `cvDeconv()` is `RefFreeCellMix_wrapper()`.

```{r IRB-RFCM2, results='hide', message=FALSE, warning=FALSE}
K=6
set.seed(1234)
outRF1 <- csDeconv(Y_raw, K, TotalIter = 30) 
```

#####3. Comparing udpated RF estimations versus RB results.

```{r IRB-RFCM3, message=FALSE, warning=FALSE}
## check the accuracy of deconvolution
estProp_RF_improved <- assignCellType(input=outRF1$estProp,
                                      reference=estProp_RB) 
mean(diag(cor(estProp_RF_improved, estProp_RB)))
```

* **A word about Step 2** 

For step 2, initial features (instead of automatic
selection by largest variation) can be provided to
function `RefFreeCellMixT()`. For example

```{r initFeature, eval = FALSE}
refinx <- findRefinx_cv(Y_raw, nmarker = 1000)
InitNames <- rownames(Y_raw)[refinx]
csDeconv(Y_raw, K = 6, nMarker = 1000, 
         InitMarker = InitNames, TotalIter = 30)
```

#### Improved-RF with use-defined RF function

In order to use other RF functions, users can 
wrap the RF function a bit first to make it 
accept Y (raw data) and K (number of cell types)
as input, and return a N (number of cell types) 
by K proportion matrix. We take `RefFreeCellMix()`
as an example. Other deconvolution methods can be
used similarly.

```{r, eval = FALSE}
mydeconv <- function(Y, K){
     outY = RefFreeEWAS::RefFreeCellMix(Y, 
               mu0=RefFreeEWAS::RefFreeCellMixInitialize(Y, 
               K = K))
     Prop0 = outY$Omega
     return(Prop0)
}
set.seed(1234)
outT <- csDeconv(Y_raw, K, FUN = mydeconv)
```



## 5. Detect cell type-specific and cross-cell type differential signals

The csDE/csDM detection function requires 
a table of microarray or RNA-seq measurements
from all samples, a table of mixing proportions, 
and a design vector representing the status of 
subjects.  

We demonstrate the usage of TOAST in three common settings.

### 5.1 Detect cell type-specific differential signals from the basic two group-designed experiment

####1. First load in the library and the dataset.

```{r csDE1}
library(TOAST)
data("RA_100samples")
```

#####2. Generate the study design based on the 
phenotype matrix. Note that all the binary 
variable (e.g. disease = 0, 1) or categorical
variable (e.g. gender = 1, 2) should be transformed
to factor class. Here we use the proportions 
estimated from step \@ref(section:RFimp) as
input proportion.

```{r csDE2}
head(Pheno, 3)
design <- data.frame(disease = as.factor(Pheno$disease))

Prop <- estProp_RF_improved
colnames(Prop) <- colnames(Ref) 
## columns of proportion matrix should have names

```

####3. Make model design using the design (phenotype)

data frame and proportion matrix. 
```{r csDE3}
Design_out <- makeDesign(design, Prop)
```

####4. Fit linear models for raw data and the 
design generated from `Design_out()`.

```{r csDE4}
fitted_model <- fitModel(Design_out, Y_raw)
# print all the cell type names
fitted_model$all_cell_types
# print all phenotypes
fitted_model$all_coefs
```

TOAST allows a number of hypotheses to be 
tested using `csTest()` in two group setting.

####5.1 Testing parameter effect (e.g. disease) in one cell type.

For example, testing disease (patient versus controls) 
effect in Gran.

```{r}
res_table <- csTest(fitted_model, 
coef = "disease", 
cell_type = "Gran")
head(res_table, 3)
Disease_Gran_res <- res_table
```

####5.2 Testing parameter effect in all cell types.

For example, testing the joint effect of age 
in all cell types:

```{r, eval = FALSE}
res_table <- csTest(fitted_model, 
coef = "disease", 
cell_type = "joint")
head(res_table, 3)
```

Specifying cell_type as NULL or not specifying 
cell_type will test the effect in each cell type
and the joint effect in all cell types.

```{r joint, eval = FALSE}
res_table <- csTest(fitted_model, 
coef = "disease", 
cell_type = NULL)
lapply(res_table, head, 3)

## this is exactly the same as
res_table <- csTest(fitted_model, 
coef = "disease")
```

## 5.2 Detect cell type-specific differential signals from the general experimental design

####1. First load in the library and the dataset.

```{r general1}
library(TOAST)
data("RA_100samples")
```

####2. Generate the study design based on the phenotype
matrix. Note that all the binary variable 
(e.g. disease = 0, 1) or categorical variable 
(e.g. gender = 1, 2) should be transformed 
to factor class. 

```{r general2}
design <- data.frame(age = Pheno$age,
                     gender = as.factor(Pheno$gender),
                     disease = as.factor(Pheno$disease))

Prop <- estProp_RF_improved
colnames(Prop) <- colnames(Ref)  ## columns of proportion matrix should have names
```

####3. Make model design using the design (phenotype)
data frame and proportion matrix. 
```{r general3}
Design_out <- makeDesign(design, Prop)
```

####4. Fit linear models for raw data and the 
design generated from `Design_out()`.

```{r general4}
fitted_model <- fitModel(Design_out, Y_raw)
# print all the cell type names
fitted_model$all_cell_types
# print all phenotypes
fitted_model$all_coefs
```

TOAST allows a number of hypotheses to be 
tested using `csTest()` in two group setting.

####5.1 Testing the effect of single parameter in one cell type.

For example, testing age effect in Gran.

```{r general5}
res_table <- csTest(fitted_model, 
coef = "age", 
cell_type = "Gran")
head(res_table, 3)
```

We can test disease effect in Bcell.

```{r general6}
res_table <- csTest(fitted_model, 
coef = "disease", 
cell_type = "Bcell")
head(res_table, 3)
```

Instead of using the names of single coefficient, 
you can specify contrast levels, i.e. the comparing
levels in this coefficient. For example, using male
(gender = 1) as reference, testing female (gender = 2)
effect in CD4T:
```{r}
res_table <- csTest(fitted_model, 
coef = c("gender", 2, 1), 
cell_type = "CD4T")
head(res_table, 3)
```

####5.2 Testing the joint effect of single parameter in all cell types.

For example, testing the joint effect of age in all cell types:

```{r, eval = FALSE}
res_table <- csTest(fitted_model, 
coef = "age", 
cell_type = "joint")
head(res_table, 3)
```

Specifying cell_type as NULL or not specifying
cell_type will test the effect in each cell type
and the joint effect in all cell types.

```{r, eval = FALSE}
res_table <- csTest(fitted_model, 
coef = "age", 
cell_type = NULL)
lapply(res_table, head, 3)

## this is exactly the same as
res_table <- csTest(fitted_model, 
coef = "age")
```


## 5.3 Detect cross-cell type differential signals

####1. First load in the library and the dataset.

```{r crossCellType1, eval = FALSE}
library(TOAST)
data("RA_100samples")
```

####2. Generate the study design based on the 
phenotype matrix. We allow general design
matrix such as the following:

```{r crossCellType2}
design <- data.frame(age = Pheno$age,
                     gender = as.factor(Pheno$gender),
                     disease = as.factor(Pheno$disease))

Prop <- estProp_RF_improved
colnames(Prop) <- colnames(Ref)  ## columns of proportion matrix should have names
```

Note that if all subjects belong to one group, 
we also allow detecting cross-cell type differences.
In this case, the design matrix can be specified as:

```{r crossCellType3, eval = FALSE}
design <- data.frame(disease = as.factor(rep(0,100)))
```

####3. Make model design using the design (phenotype) 
data frame and proportion matrix. 

```{r crossCellType4}
Design_out <- makeDesign(design, Prop)
```

####4. Fit linear models for raw data and the 
design generated from `Design_out()`.

```{r crossCellType5}
fitted_model <- fitModel(Design_out, Y_raw)
# print all the cell type names
fitted_model$all_cell_types
# print all phenotypes
fitted_model$all_coefs
```

For cross-cell type differential signal detection, 
TOAST also allows multiple ways for testing. 
For example

####5.1 Testing cross-cell type differential signals in cases (or in controls).

For example, testing the differences between 
CD8T and B cells in case group

```{r}
test <- csTest(fitted_model, 
coef = c("disease", 1), 
cell_type = c("CD8T", "Bcell"), 
contrast_matrix = NULL)
head(test, 3)
```

Or testing the differences between 
CD8T and B cells in control group

```{r}
test <- csTest(fitted_model, 
coef = c("disease", 0), 
cell_type = c("CD8T", "Bcell"), 
contrast_matrix = NULL)
head(test, 3)
```

####5.2 Testing the overall cross-cell type differences in all samples.

For example, testing the overall differences 
between Gran and CD4T in all samples, 
regardless of phenotypes.

```{r}
test <- csTest(fitted_model, 
coef = "joint", 
cell_type = c("Gran", "CD4T"), 
contrast_matrix = NULL)
head(test, 3)
```

If you do not specify `coef` but only 
the two cell types to be compared, TOAST 
will test the differences of these 
two cell types in each coef parameter 
and the overall effect.

```{r}
test <- csTest(fitted_model, 
coef = NULL, 
cell_type = c("Gran", "CD4T"), 
contrast_matrix = NULL)
lapply(test, head, 3)
```


####5.3 Testing the differences of two cell types over different values of one phenotype (higher-order test).

For example, testing the differences 
between Gran and CD4T in disease patients 
versus in controls.

```{r}
test <- csTest(fitted_model, 
coef = "disease", 
cell_type = c("Gran", "CD4T"), 
contrast_matrix = NULL)
head(test, 3)
```

For another example, testing the differences 
between Gran and CD4T in males versus females.

```{r}
test <- csTest(fitted_model, 
coef = "gender", 
cell_type = c("Gran", "CD4T"), 
contrast_matrix = NULL)
head(test, 3)
```

### A few words about variance bound and Type I error.

#### Variance bound
There is an argument in `csTest()` called 
`var_shrinkage`. `var_shrinkage` is whether
to apply shrinkage on estimated mean squared
errors (MSEs) from the regression. 
Based on our experience, extremely 
small variance estimates sometimes cause
unstable test statistics. In our implementation,
use the 10% quantile value to bound the smallest MSEs.
We recommend to use the default opinion 
`var_shrinkage = TRUE`. 

#### Type I error
For all the above tests, we implement them 
using F-test. In our own experiments, we 
observe inflated type I errors from using 
F-test. As a result, we recommend to perform
a permutation test to validate the significant
signals identified are "real". 

## References

Li, Z., et al. "Dissecting differential signals in high-throughput data from complex tissues." Bioinformatics (Oxford, England) (2019).

Gong, Ting, et al. "Optimal deconvolution of transcriptional profiling data using quadratic programming with application to complex clinical blood samples." PloS one 6.11 (2011): e27156.

Repsilber, Dirk, et al. "Biomarker discovery in heterogeneous tissue samples-taking the in-silico deconfounding approach." BMC bioinformatics 11.1 (2010): 27.

Abbas, Alexander R., et al. "Deconvolution of blood microarray data identifies cellular activation patterns in systemic lupus erythematosus." PloS one 4.7 (2009): e6098.

Shen-Orr, Shai S., et al. "Cell type–specific gene expression differences in complex tissues." Nature methods 7.4 (2010): 287.

Zhong, Yi, et al. "Digital sorting of complex tissues for cell type-specific gene expression profiles." BMC bioinformatics 14.1 (2013): 89.

Houseman, Eugene Andres, John Molitor, and Carmen J. Marsit. "Reference-free cell mixture adjustments in analysis of DNA methylation data." Bioinformatics 30.10 (2014): 1431-1439.

