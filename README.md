# GI_mapping

This repository takes paired guide RNA counts data and calculates Genetic Interaction (GI) scores

The entire analysis can be re-run on the Fred Hutch clusters if this repository is cloned there and ran from the Berger lab folders using the `run_pipeline.sh` script.

The `run_pipeline.sh` script calls a Snakemake workflow (`workflows/Snakefile`). This is the core of the analysis and does the following:

- It uses a conda environment so it shas the Python libraries and other software it needs to run. This information is from the `env/config.sample.yaml` file.
- Next, it gets the count file names from a specific directory and stores them in a list, along with the cell line names extracted from the file names.
- It defines a rule named `all` which specifies the final output files that the workflow should produce. The expand function is used to generate multiple file paths by substituting the `{cell_line}` wildcard with each cell line name in the `cell_line_list`. `{cell_line}` is a wildcard that gets replaced with actual cell line names when the workflow is run. This allows the same rule to be used for multiple cell lines.
- Next there's a series of steps defined by each of these "rules". Each of these steps has their input, output, and separate conda environment, parameters, log file, and shell command to execute that need to be specified.

Each step has these settings (I've described in plain speak what these are generally for).
```
input:
    "This builds together the input file name using the wildcards specified at the start of the file"
output:
    "This specifies where the output results files should be stored"
conda:
    "This tells us where the conda environment file is for this step so we have the packages we need"
params:
    "This is defining the files and folders and other parameters"
log:
    "Where the log should be stored"
shell:
    "A shell command to be run that has the wildcards that are defined above -- this is what is doing the work"
```
### Core steps in the workflow:

In the snakefile you can see where this is called, but if you want to see what is happening in the actual step you need to look at the corresponding Rmd file in the `scripts` folder.

### `scripts/pgRNA_counts_QC.Rmd`

This Rmd runs QC and applies a low count filter

  - It makes a cummulative distribution function
  - Prints out the Counts per million (CPM) per sample
  - Does a sample to sample correlation
  - Flags samples that maybe don't have enough counts for the plasmid
    - This requries a cutoff to be set -- how low is too low?
  - Then prints out what was removed.
- `get_pgRNA_annotations` - This annotates the data
  - It grabs Entrez Ids and Ensembl annotation
  - Grabs Copy Number and Transcript per Millions data from a cancer dependency dataset using [depMap](https://bioconductor.org/packages/release/data/experiment/vignettes/depmap/inst/doc/depmap.html#1_introduction)
  - It labels genes as `negative_control`, `positive_control`, `single_targeting` or `double_targeting`.

### `scripts/calculate_LFC.Rmd`

This Rmd calculates log fold change and makes heatmaps

  - Does filtering based on annotations
  - Uses custom plotting functions to make heatmaps and violin plots
  - Calculates  SSMD?
  - Investigates correlations across replicates
  - How log fold changed is normalized:
    - `Take LFC, then subtract median of negative controls. This will result in the median of the nontargeting being set to 0. Then, divide by the median of negative controls (double non-targeting) minus median of positive controls (targeting 1 essential gene). This will effectively set the median of the positive controls (essential genes) to -1.`
    - `Since the pgPEN library uses non-targeting controls, we adjusted for the fact that single-targeting pgRNAs generate only two double-strand breaks (1 per allele), whereas the double-targeting pgRNAs generate four DSBs. To do this, we set the median (adjusted) LFC for unexpressed genes of each group to zero.` 
  - Does different handling for single level targeting versus double level targeting
  - Calculates target level values

### `scripts/calculate_GI_scores.Rmd`

This Rmd calculates Genetic Interaction scores

  - Calculates CRISPR mean score and handles single versus double targeted genes differently
  - Creates a linear model
  - Plots the GI scores
  - Lastly a a Wilcoxon rank-sum test and t tests are performed using the calculated GI scores

### Utils

Additionally there is a script that has some util functionality that other steps borrow from:`scripts/shared_functions_and_variables.R`.


### Original README is below:

* include background on file name formatting
* define the goals of the package, include a figure
* write out all requirements for formatting files, etc. (or make a readthedocs?)

## To Do

### Phoebe
* at what stage can I take the mean across reps? Alice says keep the reps in, but add the mean as another "rep" - decide what is best to do for this
* stick with average lm method for now, add in binned median as well
* name function arguments
* make sure I get the same output I got from my original analysis
* think about Jesse's comment from my committee meeting re: calculating SL GI scores for low-scoring pairs
* consider changing target_type from gene_ctrl and ctrl_gene to gene1_only and gene2_only (or something similar) in case people have a different library design
* convert id => pgRNA_id in all relevant files (annotation for last step of pgRNA counting pipeline?)
* make Rproj? If I can figure out how to do this in a useful way with the command line - or should I use renv instead of conda?
  * https://rstudio.github.io/renv/articles/collaborating.html
* remove really big DFs from memory once they are no longer needed
* save RData somehow (but make sure I'm not saving big unnecessary datasets)
* write fxn to convert Python/bash-formatted input to R file path input?
* update counts QC Rmd to get library size dynamically
* make it so that positive controls are also expressed?
* figure out when/how to make results files (for Snakemake and Rmd)
* compare my calculated LFC values to those from MAGeCK
* figure out when to filter out low read count pgRNAs - in the counts_QC or calculate_LFC scripts?
* within pgRNA_counts_QC.Rmd
  * check CPM calculation to determine if it's correct, share w/ Daniel
* figure out if "normalized" count from MAGeCK = CPM - if so, can just do LFC calculations in my own script
* make annotations in a separate script - do for all library files, and take user input re: having RNAseq data or getting from DepMap
* set the default parameter so that there is an appropriate error message if no parameter is supplied?
* consider using `rmarkdown::render("MyDocument.Rmd", params = "ask")` (see: https://bookdown.org/yihui/rmarkdown/params-knit.html) to get user input rather than relying solely on the config file
* change sample/rep labels from days to "plasmid", "early", "late"? Or define what those are in the config file and then import that info as parameters to R scripts?
* address cases w/ multiple early TP reps
* convert rule 1 output from .html file to .txt file
* change counter_efficient.R output to make variable names just be the sample name (not counts_sampleName)
* should I filter out count = 0 before calculating CPM outliers??
* try out formatR formatting and options: https://bookdown.org/yihui/rmarkdown-cookbook/opts-tidy.html
* if I end up copying MAGeCK source code, include this info: https://github.com/davidliwei/mageck/blob/master/COPYING


#### Completed:
* write a function to print kable output ... include some kind of nrow() cutoff?
* calculate coverage in pgRNA_counts_QC Rmd
* write a function to save plots and tbls
* save functions as separate scripts & call from each Rmd using source()
* figure out how to knit Rmd with parameters => do this within the Snakemake pipeline
* update id => pgRNA_id
* pre-process annotations file to get a saveable/shareable one => update get_pgRNA_annotations.Rmd
* get rid of "d." in saved variable names...except for RDS files?
