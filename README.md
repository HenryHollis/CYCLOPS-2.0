# CYCLOPS 2.0
CYCLOPS (Cyclic Ordering by Periodic Structure) is designed to reconstruct the temporal ordering of high dimensional data generated by a common periodic process. This module is an improvement to the original CYCLOPS. The improved CYCLOPS 2.0 is written for Julia 1.6.  
  
This repository contains two '.jl' files. 'CYCLOPS.jl' contains all the functions necessary to pre-process data, train CYCLOPS 2.0, and calculate cosinors of best fit for all transcripts using CYCLOPS sample phase predictions. 'CYCLOPS_2_0_Template.jl' calls the necessary functions from 'CYCLOPS.jl' to order an expression file, given a list of seed genes and hyperparameters. The 'CYCLOPS_2_0_Template.jl' script is copied and pasted into a terminal running Julia 1.6.  
  
*Skip to '2. Packages' to learn how to use the code.*  
  
# Contents  
1. Methods  
2. Packages  
3. Expression Data File  
4. Seed Genes  
5. Covariates  
6. Hyperparameters  
7. Sample Collection Times  
8. CYCLOPS.Fit  
9. CYCLOPS.Align  
10. Contact  
   
# 1. Methods  
  
## Expression Data to eigengenes
  
Pre-processing methods are performed as described by Anafi et al. 2017$`^{[6 (1)]}`$. Probes are restricted to those in the “seed gene” list (*see 'Seed Genes'*) and the top 10,000 most highly expressed (*see ':seed_mth_Gene' in 'Hyperparameters'*). Of these, the list is further limited to those with a coefficient of variation between 0.14 and 0.9 (*see ':seed_min_CV' and ':seed_max_CV' in 'Hyperparameters'*). For these probes, extreme expression values are capped at the top/bottom 2.5th percentile (*see ':blunt_percent' in 'Hyperparameters'*). The expression $X_{i,j}$ of each included probe $i$ in sample $j$ is scaled to give $S_{i,j}$:  
  
$$\quad\quad\quad\quad    S_{i,j}=\frac{X_{i,j}-M_i}{M_i},   \quad\quad\quad\quad\dots(f1)$$
  
where $M_i$ is the mean expression of probe $i$ across samples: 
  
```math
\quad\quad\quad\quad    M_i=\Big(\frac{1}{N}\Big)\sum_j^kX_{i,j}.    \quad\quad\quad\quad\dots(f2)
```
  
The $S_{i,j}$ data are expressed in eigengene coordinates $E_{i,j}$ following the methods of Alter et al$`^[(2)]`$. The number of eigengenes $N_E$ (singular values) retained is set to capture 85% of the seed data’s total variance (*see ':eigen_total_var' in 'Hyperparameters'*), and an eigengene must contribute at least 6% of the data's total variance to be included (*see ':eigen_contr_var' in 'Hyperparameters*), as described by Anafi et al. 2017$`^{[6 (1)]}`$.  
  
## Covariate Processing
  
Discontinuous covariates are encoded into reduced one-hot flags. Reduced one-hot encoding results in an $M$-by-$`k`$ matrix, where $M$ is the sum of all covariate groups, less the number of covariates. This can be represented mathematically as:  
  
$$\quad\quad\quad\quad    M=\sum_{c=1}^C(m_c)-C   \quad\quad\quad\quad\dots(f3)$$  
  
where $C$ is the total number of covariates included, and $m$ is the number of groups in covariate $c$.  
  
## Example Covariate Processing
  
*Why use reduced one-hot encoding?* To reduce the number of free (trainable) parameters. Consider a hypothetical dataset ($`EG_1`$) with $k$ samples from two (2) batches ($`M=1`$).  
  
| Covariate | Sample$`_1`$ | Sample$`_2`$ | Sample$`_3`$ | Sample$`_4`$ | $`\dots`$ | Sample$`_k`$ |
------------|--------------|--------------|--------------|--------------|-----------|--------------|
| Batch     | B1           | B2           | B2           | B1           | $`\dots`$ | B2           |
  
In standard one-hot encoding, a sample from batch one (1) and batch two (2) are represented as  
  
```math
\begin{bmatrix}
1 \\[0.3em]
0
\end{bmatrix}\ \&
\begin{bmatrix}
0 \\[0.3em]
1
\end{bmatrix},\ respectively.
```  
  
This notation contains redundant information. The first row of the matrix can be ommitted without any loss of information, resulting in reduced one-hot encodings for samples in batch one (1) and two (2):  
  
$$[0]\ \\&\ [1],\ respectively.$$  
  
This reduces the number of free parameters in a model by the number of eigengenes $`N_E`$ retained for a particular dataset (*see 'CYCLOPS 2.0 Model'*).  
  
Now, consider a hypothetical dataset ($`EG_2`$) with $k$ samples from four (4) batches, where samples could be from one of two (2) tissue types ($`M=4`$).  
  
| Covariate | Sample$`_1`$ | Sample$`_2`$ | Sample$`_3`$ | Sample$`_4`$ | $`\dots`$ | Sample$`_N`$ |
------------|--------------|--------------|--------------|--------------|-----------|--------------|
| Batch     | B1           | B2           | B3           | B4           | $`\dots`$ | B3           |
| Type      | T1           | T2           | T1           | T2           | $`\dots`$ | T2           |  
  
Instead of considering these two covariates as a combination of states, which would result in eight possible states...
  
```math
\begin{bmatrix}
Batch_1\ \&\ Type_1 \\[0.3em]
Batch_2\ \&\ Type_1 \\[0.3em]
Batch_3\ \&\ Type_1 \\[0.3em]
Batch_4\ \&\ Type_1 \\[0.3em]
Batch_1\ \&\ Type_2 \\[0.3em]
Batch_2\ \&\ Type_2 \\[0.3em]
Batch_3\ \&\ Type_2 \\[0.3em]
Batch_4\ \&\ Type_2
\end{bmatrix}
```
...we consider each covariate independently.
  
```math
\begin{bmatrix}
Batch_1 \\[0.3em]
Batch_2 \\[0.3em]
Batch_3 \\[0.3em]
Batch_4
\end{bmatrix}\ can\ be\ reduced\ to\ 
\begin{bmatrix}
Batch_2 \\[0.3em]
Batch_3 \\[0.3em]
Batch_4
\end{bmatrix}
```
and
```math
\begin{bmatrix}
Type_1 \\[0.3em]
Type_2
\end{bmatrix}\ can\ be\ reduced\ to\ 
\begin{bmatrix}
Type_2
\end{bmatrix}.
```  
CYCLOPS usese the combined reduced one-hot encodings. 
```math
\begin{bmatrix}
Batch_2 \\[0.3em]
Batch_3 \\[0.3em]
Batch_4 \\[0.3em]
Type_2
\end{bmatrix}.
```  
  
CYCLOPS 2.0 was optimized to fit the eigengene expression, including each sample's reduced one-hot covariate encoding.  
  
## CYCLOPS 2.0 Model  
  
The CYCLOPS 2.0 Model comprises the core structure and the reduced one-hot encoded layers. The core structure is the original CYCLOPS structure described in Anafi et al. 2017$`^{[6 (1)]}`$.  
  
```math
{\color{Goldenrod}input\ data}\ \to\ {\color{Cyan}reduced\ one\text{-}hot\ encoding}\ \to\ {\color{Red}core\ CYCLOPS\ structure}\ \to\ {\color{Cyan}reduced\ one\text{-}hot\ decoding}
```  
  
### Encoding Layer  
The reduced one-hot encoded layers consist of an encoding and decoding side. The encoding and decoding reduced one-hot layers share the same parameters, only use linear transformations, and are functional inverses.  
  
```math  
{\color{Cyan}reduced\ one\text{-}hot\ encoded\ data}={\color{Goldenrod}input\ data}∘(1+W∙G)+b∙C+d.
```  
    
### Decoding Layer
```math  
{\color{Cyan}reduced\ one\text{-}hot\ decoded\ data}=
\frac{{\color{Red}core\ CYCLOPS\ output}-(b∙C+d)}{1+W∙G}
```  
  
### Parameters
- “$`∘`$” is the element-wise matrix product,  
- and “$`∙`$” is the matrix dot product.  
- The input data for sample $j$ is a column vector with $N_E$ rows.  
- $G$ is the covariate encoding of sample $j$, and a column vector with $M$ rows.  
- $W$ is an $N_E$-by-$`M`$ (rows-by-columns) matrix.  
- Similarly, $b$ is also an $N_E$-by-$`M`$ matrix.  
- $d$ is a column vector with $N_E$ rows.
  
The resulting covariate encoded and decoded data have the exact dimensions as the input data.
  
### Example Covariate Encoding
Consider $`EG_2`$, then $`M=4`$. Lets assume $`N_E=5`$.  
  
```math
{\color{Goldenrod}input\ data}=
\begin{bmatrix}
1 \\[0.3em]
2 \\[0.3em]
3 \\[0.3em]
4 \\[0.3em]
5
\end{bmatrix}
```  
  
# 2. Packages
This module requires the following packages (- version):  
```
CSV --------------- v0.10.4  
DataFrames -------- v1.3.4  
Distributions ----- v0.25.68  
Flux -------------- v0.13.5  
MultipleTesting --- v0.5.1  
MultivariateStats - v0.10.0  
Plots ------------- v1.38.11  
PyPlot ------------ v2.11.0  
Revise ------------ v3.4.0  
StatsBase --------- v0.33.21  
XLSX -------------- v0.8.4  
Dates  
Distributed  
LinearAlgebra  
Random  
Statistics  
```
  
# 3. Expression Data File
The expression data are a required input to the *CYCLOPS.Fit* and *CYCLOPS.Align* functions. The format of the expression data file is as follows:  
  
1. Each column is a sample.
2. Each row is a transcript.
3. The first column of the dataset contains gene symbols and covariate labels (see 'Covariates' and 'Hyperparameters').
4. The first 'k' rows contain covariates and all following rows contain numeric expression data (see 'Covariates').
5. Samples and columns with 'missing' or 'NaN' values should be removed from the dataset.
6. Column names start with letters and only contain numbers, letters, and underscores ('_').
7. All Gene symbols should start with letters.
8. Duplicate gene symbols **are** allowed.
9. Duplicate column names are **NOT** allowed.
  
## Example Expression Data File
```
12869×653 DataFrame
   Row │ Gene_Symbol   GTEX_1117F_2826_SM_5GZXL  GTEX_1122O_1226_SM_5H113  GTEX ⋯
───────┼─────────────────────────────────────────────────────────────────────────
     1 │ tissueType_D  NonTumor                  Tumor                     NonT ⋯
     2 │ site_D        B1                        B1                        B1
     3 │ WASH7P        10.04                     3.248                     4.82
     4 │ LINC01128     5.357                     7.199                     4.57
     5 │ SAMD11        0.6739                    1.213                     0.46 ⋯
     6 │ NOC2L         64.54                     62.24                     73.7
     7 | NOC2L         65.52                     61.18                     74.8
   ⋮   |      ⋮                   ⋮                         ⋮                   ⋱
 12865 │ MTCP1         6.513                     6.308                     7.13
 12866 │ BRCC3         9.685                     10.12                     16.1
 12867 │ VBP1          34.65                     30.77                     25.4 ⋯
 12868 │ CLIC2         21.85                     30.05                     16.8
 12869 │ TMLHE         5.04                      4.06                      5.24
```
  
# 4. Seed Genes  
The seed genes are a required input to the *CYCLOPS.Fit* function. Seed genes must be provided as a vector of strings (not symbols). Also consider:  
  
1. Case matters!  
   *"Acot4" in the seed gene list will not match "ACOT4" in the gene symbols of the expression data.*  
2. Provide enough seed genes.  
   *The number of seed genes should be greater than ':eigen_max' (see 'Hyperparameters').*  
3. Seed genes start with letters.  
   *Gene symbols in the expression data should start with letters, therefore seed genes should also start with letters.*  
  
## Example Seed Genes
```
71-element Vector{String}:
 "ACOT4"
 "ACSM5"
 "ADORA1"
 "ADRB3"
 "ALAS1"
 "ANGPTL2"
 "ARHGAP20"
 "ARNTL"
 ⋮
 "TP53INP2"
 "TSC22D3"
 "TSPAN4"
 "TUSC5"
 "USP2"
 "WEE1"
 "ZEB2"
```
  
# 5. Covariates  
The expression data may contain rows of grouping variables (discontinuous covariates) or continuous variables (continuous covariates). The following constraints apply:  
  
1. All covariate rows must be above expression data.
2. Grouping variables (discontinous covariates) must start with letters.  
3. Continuous variables (continuous covariates) must **only** contain numbers.  
4. Within a row of grouping variables (discontinuous covariates), each group must contain at least two (2) or more samples.  
   *Consider sample tissue type as a covariate. If the data has 'Non Tumor' and 'Tumor' samples, there should be at least two (2) 'Non Tumor' and two (2) 'Tumor' samples in the data. Ideally, the number of samples in each group is greater than ':eigen_max' (see 'Hyperparameters').*  
5. All samples must have have values for **all** covariates (rows).  
6. Covariate rows must have unique names.  
7. Covariate rows have regex identifiers <ins>in the gene symbol column</ins>.  
   *The example below, discontinuous covariates end in '_D' and continuous covariates end in '_C'. See 'Hyperparameters.'*  
  
## Example Covariates
Below is a sample dataset with three (3) covariates and two (2) genes. Two (2) covariates are discontinuous ('tissueType_D' and 'site_D'), and one (1) is continuous ('age_C').
```
   Row │ Gene_Symbol   GTEX_1117F_2826_SM_5GZXL  GTEX_1122O_1226_SM_5H113  GTEX ⋯
───────┼─────────────────────────────────────────────────────────────────────────
     1 │ tissueType_D  NonTumor                  Tumor                     NonT ⋯
     2 │ site_D        B1                        B1                        B1
     3 │ age_C         25                        56                        62
     4 │ GENE1         4.32                      27.63                     18.43
     5 │ GENE2         17.58                     21.42                     35.67
```
Note that the example expression data ('GENE1' & 'GENE2') are below all covariate rows (rows 1-3). No other covariates (ending in '_D' or '_C') should be present below 'GENE1.'  
  
# 6. Hyperparameters  
Hyperparameters are stored in a single dictionary to reduce the number of individual input arguments to the *CYCLOPS.Fit* and *CYCLOPS.Align* functions. Below is the default hyperparameter dictionary, including default values. It is not recommended to alter default values unless absolutely necessary. Any changes to the values of the hyperparameter dictionary will result in warnings printed to the REPL when running the *CYCLOPS.Fit* function.
  
## Hyperparameters with Default Values
```julia
Dict(  
  :regex_cont => r".*_C",                # What is the regex match for continuous covariates in the data file?
  :regex_disc => r".*_D",                # What is the regex match for discontinuous covariates in the data file?

  :blunt_percent => 0.975,               # What is the percentile cutoff below (lower) and above (upper) which values are capped?

  :seed_min_CV => 0.14,                  # The minimum coefficient of variation a gene of interest may have to be included in eigengene transformation
  :seed_max_CV => 0.7,                   # The maximum coefficient of a variation a gene of interest may have to be included in eigengene transformation
  :seed_mth_Gene => 10000,               # The minimum mean a gene of interest may have to be included in eigengene transformation

  :norm_gene_level => true,              # Does mean normalization occur at the seed gene level
  :norm_disc => false,                   # Does batch mean normalization occur at the seed gene level
  :norm_disc_cov => 1,                   # Which discontinuous covariate is used to mean normalize seed level data

  :eigen_reg => true,                    # Does regression against a covariate occur at the eigengene level
  :eigen_reg_disc_cov => 1,              # Which discontinous covariate is used for regression
  :eigen_reg_exclude => false,           # Are eigengenes with r squared greater than cutoff removed from final eigen data output
  :eigen_reg_r_squared_cutoff => 0.6,    # This cutoff is used to determine whether an eigengene is excluded from final eigen data used for training
  :eigen_reg_remove_correct => false,    # Is the first eigengene removed (true --> default) or it's contributed variance corrected by batch regression (false)

  :eigen_first_var => false,             # Is a captured variance cutoff on the first eigengene used
  :eigen_first_var_cutoff => 0.85,       # Cutoff used on captured variance of first eigengene

  :eigen_total_var => 0.85,              # Minimum amount of variance required to be captured by included dimensions of eigengene data
  :eigen_contr_var => 0.06,              # Minimum amount of variance required to be captured by a single dimension of eigengene data
  :eigen_var_override => false,          # Is the minimum amount of contributed variance ignored
  :eigen_max => 30,                      # Maximum number of dimensions allowed to be kept in eigengene data

  :out_covariates => true,               # Are covariates included in eigengene data
  :out_use_disc_cov => true,             # Are discontinuous covariates included in eigengene data
  :out_all_disc_cov => true,             # Are all discontinuous covariates included if included in eigengene data
  :out_disc_cov => 1,                    # Which discontinuous covariates are included at the bottom of the eigengene data, if not all discontinuous covariates
  :out_use_cont_cov => false,            # Are continuous covariates included in eigen data
  :out_all_cont_cov => true,             # Are all continuous covariates included in eigengene data
  :out_use_norm_cont_cov => false,       # Are continuous covariates Normalized
  :out_all_norm_cont_cov => true,        # Are all continuous covariates normalized
  :out_cont_cov => 1,                    # Which continuous covariates are included at the bottom of the eigengene data, if not all continuous covariates, or which continuous covariates are normalized if not all
  :out_norm_cont_cov => 1,               # Which continuous covariates are normalized if not all continuous covariates are included, and only specific ones are included

  :init_scale_change => true,            # Are scales changed
  :init_scale_1 => false,                # Are all scales initialized such that the model sees them all as having scale 1
                                         # Or they'll be initialized halfway between 1 and their regression estimate.

  :train_n_models => 80,                 # How many models are being trained
  :train_μA => 0.001,                    # Learning rate of ADAM optimizer
  :train_β => (0.9, 0.999),              # β parameter for ADAM optimizer
  :train_min_steps => 1500,              # Minimum number of training steps per model
  :train_max_steps => 2050,              # Maximum number of training steps per model
  :train_μA_scale_lim => 1000,           # Factor used to divide learning rate to establish smallest the learning rate may shrink to
  :train_circular => false,              # Train symmetrically
  :train_collection_times => true,       # Train using known times
  :train_collection_time_balance => 0.1, # How is the true time loss rescaled

  :cosine_shift_iterations => 192,       # How many different shifts are tried to find the ideal shift
  :cosine_covariate_offset => true,      # Are offsets calculated by covariates

  :align_p_cutoff => 0.05,               # When aligning the acrophases, what genes are included according to the specified p-cutoff
  :align_base => "radians",              # What is the base of the list (:align_acrophases or :align_phases)? "radians" or "hours"
  :align_disc => false,                  # Is a discontinuous covariate used to align (true or false)
  :align_disc_cov => 1,                  # Which discontinuous covariate is used to choose samples to separately align (is an integer)
  :align_other_covariates => false,      # Are other covariates included
  :align_batch_only => false,

  :X_Val_k => 10,                        # How many folds used in cross validation.
  :X_Val_omit_size => 0.1)               # What is the fraction of samples left out per fold
```
  
## Hyperparameters without Default Values
Some parameters do not have default values. These parameters come in pairs, meaning that if one parameter is added to the hyperparameter dictionary, the other parameter must also be added.  
```julia
# Pair 1
:train_sample_id    # Array{Symbol, 1}. Sample ids for which collection times exist.  
:train_sample_phase # Array{Number, 1}. Collection times for each sample id. 'train_sample_id' and 'train_sample_phase' must be the same length.  
  
# Pair 2
:align_samples      # Array{String, 1}. Sample ids for which collection times exist.  
:align_phases       # Array{Number, 1}. Collection times for each sample id. 'align_samples' and 'align_phases' must be the same length.  
  
# Pair 3
:align_genes        # Array{String, 1}. Genes with known acrophases.  
:align_acrophases   # Array{Number, 1}. Acrophases for each gene. 'align_genes' and 'align_acrophases' must be the same length.  
```
  
# 7. Sample Collection Times  
Sample collection times may be provided and added to the hyperparameter dictionary. These may be used in two different ways:
1. Semi-supervised training.  
   *CYCLOPS optimizes the distance between predicted and provided sample collection times **while training** when sample ids and collection times are added via ':train_sample_id' and ':train_sample_phase'.*  
2. Post-training alignment.  
   *CYCLOPS predicted phases are aligned to sample collection times **after training** when added via ':align_samples' and ':algin_phases'.*  
  
':train_sample_phase' **must** be given in radians ($0 - 2\pi$), **NOT** hours. By default, ':align_phases' and ':align_acrophases' must be given in radians, but should you wish to provide hours, also set ':align_base => "hours"' in the hyperparameter dictionary (see 'Hyperparameters').  

# 8. CYCLOPS.Fit  
*CYCLOPS.Fit* has three (3) input arguments and five (5) outputs.
  
## Input Arguments
1. The expression data (as described in section 3),  
2. the seed genes (as described in section 4),  
3. and the hyperparameter dictionary (as described in section 6).  
  
## Outputs  
1. The eigen data,  
2. the sample fit,  
3. the eigengene correlations,  
4. the trained model,  
5. and the updated hyperparameter dictionary.  
  
## Example Usage of CYCLOPS.Fit  
*CYCLOPS.Fit* may be used with a hyperparameter dictionary...
```julia
eigendata, samplefit, eigendatacorrelations, trainedmodel, updatedparameters = CYCLOPS.Fit(expressiondata, seedgenes, hyperparameters)
```
or without adding a hyperparameter dictionary. 
```julia
eigendata, samplefit, eigendatacorrelations, trainedmodel, updatedparameters = CYCLOPS.Fit(expressiondata, seedgenes)
```
When no hyperparameter dictionary is supplied as the third argument to *CYCLOPS.Fit*, the default values for all hyperparameters are used. Inspect the default hyperparameters using *CYCLOPS.DefaultDict*.
```julia
CYCLOPS.DefaultDict()
```
# 9. CYCLOPS.Align  
*CYCLOPS.Align* has six (6) input arguments and saves files to a directory (no outputs).  
  
## Inputs
1. the expression data (as described in section 3),  
2. the sample fit (second output of CYCLOPS.Fit),  
3. the eigengene correlations (third output of CYCLOPS.Fit),  
4. the trained model (fourth output of CYCLOPS.Fit),  
5. the updated hyperparameter dictionary (fifth output of CYCLOPS.Fit),  
6. and the output path where results are saved.  
  
## Saved Files  
*CYCLOPS.Align* creates four (4) subdirectories:  
1. Models  
   *The CYCLOPS model with the lowest reconstruction*
2. Parameters  
   *The hyperparameter dictionary*
3. Plots  
   *Clock face and sample alignment plots*
4. Fits  
   *Sample phase predictions, eigengene correlations, and cosinor regression results*
  
## Example Usage of CYCLOPS.Align  
  
```julia
CYCLOPS.Align(expressiondata, samplefit, eigendatacorrelations, trainedmodel, updatedparameters, outputpath)
```
  
## Why 'Align?'
CYCLOPS returns a relative ordering of all samples. Since a circle has no beginning, endpoint, or inherent direction, the raw predicted sample phases must in some way be aligned to the external world. There are three (3) possible ways to align the raw predicted sample phases:
1. to 17 core clock genes,
2. to genes of choice,
3. to known sample collection times.
  
The raw predicted sample phases are always aligned to the 17 core clock genes.
```julia
CYCLOPS.human_homologue_gene_symbol
CYCLOPS.mouse_acrophases
```
Using the raw predicted sample phases, cosinors of best fit are calculated for the 17 core clock genes. The cosinors of best fit tell us the predicted times at which transcripts are maximally expressed (transcript acrophases). The predicted transcript acrophases are aligned to 'CYCLOPS.mouse_acrophases.' Notably, predicted transcript acrophases are only compared for transcripts with a significant cosinor fit ($p < 0.05$).  
  
Genes of choice may be provided using ':align_genes' and ':align_acrophases' (see 'Hyperparameters'). As above, raw predicted sample phases are used to calculate cosinors of best fit, this time for the genes of choice. Predicted acrophases are aligned to ':align_acrophases' for transcripts with a significant fit ($p < 0.05$). *Please provide ':align_acrophases' in radians if ':align_base => "radians".'* (see 'Hyperparameters')  
  
If sample collection times are known for all (or a subset of) samples, these may be provided using ':align_samples' and ':align_phases' (see 'Hyperparameters'). Raw predicted sample phases are aligned to the provided known sample collection times. *Please provide ':align_phases' in radians if ':align_base => "radians".'* (see 'Hyperparameters')  
  
# 10. Contact
Please contact janham@pennmedicine.upenn.edu with questions. Kindly make the email's subject '***GitHub CYCLOPS 2.0***'. Please copy and paste the full error in the body of the email and provide the versions of all required packages. Further, provide the script and all necessary files to recreate the error you are encountering. Good luck and happy ordering.
