---
title: "Fitness vs Performance Analysis (AGS paper I)"
author: "[John Zobolas](https://github.com/bblodfon)"
date: "Last updated: 09 December, 2020"
description: "An investigation analysis"
url: 'https\://druglogics.github.io/gitsbe-model-analysis/cascade/fit-perf-ags/main.html'
github-repo: "druglogics/gitsbe-model-analysis"
link-citations: true
site: bookdown::bookdown_site
---

# Intro {-}

:::{.green-box}
Main question: is there a correlation between models fitness to the steady state and their performance?
:::

All boolean model simulations were done using the `druglogics-synergy` Java module, version `1.2.0`: `git checkout v1.2.0`.

# Input {-}

Load libraries:

```r
library(xfun)
library(emba)
library(usefun)
library(dplyr)
library(tibble)
library(stringr)
library(ggpubr)
library(ggplot2)
library(Ckmeans.1d.dp)
library(PRROC)
```

Load the AGS steady state:

```r
# get the AGS steady state
steady_state_file = "data/steadystate"
lines = readLines(steady_state_file)
ss_data = unlist(strsplit(x = lines[8], split = "\t"))
ss_mat = stringr::str_split(string = ss_data, pattern = ":", simplify = TRUE)
colnames(ss_mat) = c("nodes", "states")
ss_tbl = ss_mat %>% as_tibble() %>% mutate_at(vars(states), as.integer)

steady_state = ss_tbl %>% pull(states)
names(steady_state) = ss_tbl %>% pull(nodes)

usefun::pretty_print_vector_names_and_values(vec = steady_state)
```

> CASP3: 0, CASP8: 0, CASP9: 0, FOXO_f: 0, RSK_f: 1, CCND1: 1, MYC: 1, RAC_f: 1, JNK_f: 0, MAPK14: 0, AKT_f: 1, MMP_f: 1, PTEN: 0, ERK_f: 1, KRAS: 1, PIK3CA: 1, S6K_f: 1, GSK3_f: 0, TP53: 0, BAX: 0, BCL2: 1, CTNNB1: 1, TCF7_f: 1, NFKB_f: 1

Load observed synergies (the *Gold Standard* for the AGS cell line, as it was found by complete agreement of all curators [here](http://tiny.cc/sintefGS)):

```r
# get observed synergies for Cascade 2.0
observed_synergies_file = "data/observed_synergies_cascade_2.0"
observed_synergies = emba::get_observed_synergies(observed_synergies_file)

pretty_print_vector_values(vec = observed_synergies, vector.values.str = "observed synergies")
```

> 6 observed synergies: AK-BI, 5Z-PI, PD-PI, BI-D1, PI-D1, PI-G2

# Flip training data analysis {-}

The idea here is to generate many training data files from the steady state, where some of the nodes will have their states *flipped* to the opposite state ($0$ to $1$ and vice versa).
That way, we can train models to different steady states, ranging from ones that differ to just a few nodes states up to a steady state that is the complete *reversed* version of the one used in the simulations.

Using the [gen_training_data.R](https://github.com/druglogics/gitsbe-model-analysis/blob/master/cascade/fit-vs-performance-ags/data/gen_training_data.R) script, we first chose a few number of flips ($11$ flips) ranging from $1$ to $24$ (all nodes) in the steady state.
Then, for each such *flipping-nodes* value, we generated $20$ new steady states with a randomly chosen set of nodes whose value is going to flip.
Thus, in total, $205$ training data files were produced ($205 = 9 \times 20 + 24 + 1$, where from the $11$ number of flips, the one flip happens for every node ($24$ different steady states) and flipping all the nodes generates $1$ completely reversed steady state). 

Running the script [run_druglogics_synergy_training.sh](https://raw.githubusercontent.com/druglogics/gitsbe-model-analysis/master/cascade/fit-vs-performance-ags/data/run_druglogics_synergy_training.sh) from the `druglogics-synergy` repository root (version `1.2.0`: `git checkout v1.2.0`), we get the simulation results for each of these training data files.
We mainly going to use the **MCC performance score** for each model in the subsequent analyses.
Note that in the CASCADE 2.0 configuration file (`config`) we changed the number of simulations to ($15$) for each training data file, the attractor tool used was `biolqm_stable_states` and the `synergy_method: hsa`.

:::{.orange-box}
For the generated training data files (`training-data-files.tar.gz`) and the results from the simulations (`fit-vs-performance-results.tar.gz`) see [![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.3972340.svg)](https://doi.org/10.5281/zenodo.3972340).
:::

To load the data, download the results (`fit-vs-performance-results.tar.gz`) and extract it to a directory of your choice with the following commands: `mkdir fit-vs-performance-results` and `tar -C fit-vs-performance-results/ -xzvf fit-vs-performance-results.tar.gz`.
Then use the next `R` code to create the `res` data.frame (I have already saved the result):


```r
# change appropriately
data_dir = "/home/john/tmp/ags_paper_res/fit-vs-performance-results"
res_dirs = list.files(data_dir, full.names = TRUE)

data_list = list()
index = 1
for (res_dir in res_dirs) {
  model_predictions_file = list.files(path = res_dir, pattern = "model_predictions.tab", full.names = TRUE)
  models_dir = paste0(res_dir, "/models")
  
  model_predictions = emba::get_model_predictions(model_predictions_file)
  models_link_operators = emba::get_link_operators_from_models_dir(models_dir)
  # you get messages for models with {#stable states} != 1
  models_stable_states = emba::get_stable_state_from_models_dir(models_dir)
  
  # same order => same model per row (also pruned if != 1 stable states)
  model_predictions = model_predictions[rownames(models_stable_states),]
  models_link_operators = models_link_operators[rownames(models_stable_states),]
  
  # calculate models MCC scores
  models_mcc = calculate_models_mcc(
    observed.model.predictions = model_predictions %>% select(all_of(observed_synergies)),
    unobserved.model.predictions = model_predictions %>% select(-all_of(observed_synergies)),
    number.of.drug.comb.tested = ncol(model_predictions))
  
  # calculate models fitness to AGS steady state
  models_fit = apply(models_stable_states[, names(steady_state)], 1, 
    usefun::get_percentage_of_matches, steady_state)
  
  # check data consistency
  stopifnot(all(names(models_mcc) == rownames(models_link_operators)))
  stopifnot(all(names(models_mcc) == names(models_fit)))
  
  # get link operator mutation code per model
  # same code => exactly same link operator mutations => exactly the same model
  link_mutation_binary_code = unname(apply(models_link_operators, 1, paste, collapse = ""))
  
  # bind all to one (OneForAll)
  df = bind_cols(model_code = link_mutation_binary_code, mcc = models_mcc, fitness = models_fit)
  
  data_list[[index]] = df
  index = index + 1
}

res = bind_rows(data_list)

# prune to unique models
res = res %>% distinct(model_code, .keep_all = TRUE)

saveRDS(object = res, file = "data/res_flip.rds")
```

We split the models from the simulations to $10$ fitness classes, their centers being:


```r
res = readRDS(file = "data/res_flip.rds")

### Fitness classes vs MCC
fit_classes = res %>% pull(fitness) %>% unique() %>% length()
fit_result = Ckmeans.1d.dp(x = res$fitness, k = round(fit_classes/2))

fit_result$centers
```

```
##  [1] 0.2192911 0.3188087 0.3915344 0.4777328 0.5605536 0.6489758 0.7301299
##  [8] 0.8141840 0.8982254 0.9784706
```


```r
fitness_class_id = fit_result$cluster
res = res %>% add_column(fitness_class_id = fitness_class_id)
fit_stats = desc_statby(res, measure.var = "mcc", grps = c("fitness_class_id"), ci = 0.95)

ggbarplot(fit_stats, title = "Fitness Class vs Performance (MCC)",
  x = "fitness_class_id", y = "mean", xlab = "Fitness Class",
  ylab = "Mean MCC", ylim = c(min(fit_stats$mean - 2*fit_stats$se), max(fit_stats$mean) + 0.1),
  label = fit_stats$length, lab.vjust = -5,  fill = "fitness_class_id") + 
  scale_x_discrete(labels = round(fit_stats$fitness, digits = 2)) + 
  scale_fill_gradient(low = "lightblue", high = "red", name = "Fitness Class", 
    guide = "legend", breaks = c(1,4,7,10)) + 
  geom_errorbar(aes(ymin=mean-2*se, ymax=mean+2*se), width = 0.2, position = position_dodge(0.9))
```

<img src="index_files/figure-html/figures-1-1.png" width="2100" style="display: block; margin: auto;" />

```r
ggboxplot(res, x = "fitness_class_id", y = "mcc", color = "fitness_class_id")
```

<img src="index_files/figure-html/figures-1-2.png" width="2100" style="display: block; margin: auto;" />

```r
# useful
#res %>% group_by(fitness_class_id) %>% summarise(avg = mean(mcc), count = n()) %>% print(n = 21)
```

:::{.orange-box}
No correlation whatsoever between models fitness and performance (as measured by the MCC score)
:::

Note though that **the results are very dependent on the dataset and the observed synergies** used.
For example, using the following observed synergies vector (we also show the previous one for comparison purposes - **3 out of 6 synergies remain the same**):


```r
# get observed synergies for Cascade 2.0 (new)
observed_synergies_file = "data/observed_synergies"
observed_synergies_2 = emba::get_observed_synergies(observed_synergies_file)

pretty_print_vector_values(vec = observed_synergies, vector.values.str = "GS observed synergies")
```

> 6 GS observed synergies: AK-BI, 5Z-PI, PD-PI, BI-D1, PI-D1, PI-G2

```r
pretty_print_vector_values(vec = observed_synergies_2, vector.values.str = "New observed synergies")
```

> 6 New observed synergies: AK-BI, 5Z-D1, AK-D1, BI-D1, PI-D1, PK-ST

New scores (already saved):

```r
# change appropriately
data_dir = "/home/john/tmp/ags_paper_res/fit-vs-performance-results"
res_dirs = list.files(data_dir, full.names = TRUE)

data_list = list()
index = 1
for (res_dir in res_dirs) {
  model_predictions_file = list.files(path = res_dir, pattern = "model_predictions.tab", full.names = TRUE)
  models_dir = paste0(res_dir, "/models")
  
  model_predictions = emba::get_model_predictions(model_predictions_file)
  models_link_operators = emba::get_link_operators_from_models_dir(models_dir)
  # you get messages for models with {#stable states} != 1
  models_stable_states = emba::get_stable_state_from_models_dir(models_dir)
  
  # same order => same model per row (also pruned if != 1 stable states)
  model_predictions = model_predictions[rownames(models_stable_states),]
  models_link_operators = models_link_operators[rownames(models_stable_states),]
  
  # calculate models MCC scores
  models_mcc = calculate_models_mcc(
    observed.model.predictions = model_predictions %>% select(all_of(observed_synergies_2)),
    unobserved.model.predictions = model_predictions %>% select(-all_of(observed_synergies_2)),
    number.of.drug.comb.tested = ncol(model_predictions))
  
  # calculate models fitness to AGS steady state
  models_fit = apply(models_stable_states[, names(steady_state)], 1, 
    usefun::get_percentage_of_matches, steady_state)
  
  # check data consistency
  stopifnot(all(names(models_mcc) == rownames(models_link_operators)))
  stopifnot(all(names(models_mcc) == names(models_fit)))
  
  # get link operator mutation code per model
  # same code => exactly same link operator mutations => exactly the same model
  link_mutation_binary_code = unname(apply(models_link_operators, 1, paste, collapse = ""))
  
  # bind all to one (OneForAll)
  df = bind_cols(model_code = link_mutation_binary_code, mcc = models_mcc, fitness = models_fit)
  
  data_list[[index]] = df
  index = index + 1
}

res = bind_rows(data_list)

# prune to unique models
res = res %>% distinct(model_code, .keep_all = TRUE)

saveRDS(object = res, file = "data/res_flip_2.rds")
```

New figures:

```r
res = readRDS(file = "data/res_flip_2.rds")

### Fitness classes vs MCC
fit_classes = res %>% pull(fitness) %>% unique() %>% length()
fit_result = Ckmeans.1d.dp(x = res$fitness, k = round(fit_classes/2))
fitness_class_id = fit_result$cluster
res = res %>% add_column(fitness_class_id = fitness_class_id)
fit_stats = desc_statby(res, measure.var = "mcc", grps = c("fitness_class_id"), ci = 0.95)

ggbarplot(fit_stats, title = "Fitness Class vs Performance (MCC)",
  x = "fitness_class_id", y = "mean", xlab = "Fitness Class",
  ylab = "Mean MCC", ylim = c(min(fit_stats$mean - 2*fit_stats$se), max(fit_stats$mean) + 0.1),
  label = fit_stats$length, lab.vjust = -5,  fill = "fitness_class_id") + 
  scale_x_discrete(labels = round(fit_stats$fitness, digits = 2)) + 
  scale_fill_gradient(low = "lightblue", high = "red", name = "Fitness Class", 
    guide = "legend", breaks = c(1,4,7,10)) + 
  geom_errorbar(aes(ymin=mean-2*se, ymax=mean+2*se), width = 0.2, position = position_dodge(0.9))
```

<img src="index_files/figure-html/figures-1-new-1.png" width="2100" style="display: block; margin: auto;" />

```r
ggboxplot(res, x = "fitness_class_id", y = "mcc", color = "fitness_class_id")
```

<img src="index_files/figure-html/figures-1-new-2.png" width="2100" style="display: block; margin: auto;" />

Well, now there is correlation (but the observed synergies have been cherry-picked to get this result - so be careful!).

# Reverse SS vs SS method {-}

Next, I tried increasing the simulation output data and run only the *extreme* cases. 
So the next results are for a total of $5000$ simulations ($15000$ models), fitting to steady state (**ss**), the complete opposite/flipped steady state (**rev**) and also the [proliferation profile](https://druglogics.github.io/druglogics-doc/training-data.html#unperturbed-condition---globaloutput-response) (**prolif**).
I ran these simulations 3 times, one with **link operator mutations only, one with topology mutations only and one with both mutations** applied in the models.
The drabme synergy method used was `HSA`.
For the link operator models the `biolqm_stable_states` *attractor_tool* option was used and for the other 2 parameterizations the `bnet_reduction_reduced`.
And of course, the proper/initial observed synergies vector was used as the gold standard.

First download the dataset [![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.3972340.svg)](https://doi.org/10.5281/zenodo.3972340) and then extract the `5000sim-hsa-*-res.tar.gz` compressed archives in a directory.

Load the data using the following `R` code (I have already saved the results):

```r
# simulation results directories (do not add the ending path character '/')
res_dirs = c(
  "/home/john/tmp/ags_paper_res/topo-and-link/5000-sim-hsa-res/cascade_2.0_rev_5000sim_bnet_hsa_20200601_054944/", 
  "/home/john/tmp/ags_paper_res/topo-and-link/5000-sim-hsa-res/cascade_2.0_rand_5000sim_bnet_hsa_20200601_155644/",
  "/home/john/tmp/ags_paper_res/topo-and-link/5000-sim-hsa-res/cascade_2.0_ss_5000sim_bnet_hsa_20200531_202847/")

res_dirs = c(
  "/home/john/tmp/ags_paper_res/topology-only/5000-sim-hsa-res/cascade_2.0_rev_5000sim_bnet_hsa_20200530_153927/", 
  "/home/john/tmp/ags_paper_res/topology-only/5000-sim-hsa-res/cascade_2.0_rand_5000sim_bnet_hsa_20200531_014742/",
  "/home/john/tmp/ags_paper_res/topology-only/5000-sim-hsa-res/cascade_2.0_ss_5000sim_bnet_hsa_20200531_080616/")

data_list = list()
index = 1
for (res_dir in res_dirs) {
  if (str_detect(pattern = "cascade_2.0_ss", string = res_dir)) {
    type = "ss" 
  } else if (str_detect(pattern = "cascade_2.0_rev", string = res_dir)) {
    type = "rev" 
  } else {
    type = "prolif" 
  }
  
  model_predictions_file = list.files(path = res_dir, pattern = "model_predictions.tab", full.names = TRUE)
  models_dir = paste0(res_dir, "/models")
  
  model_predictions = emba::get_model_predictions(model_predictions_file)
  # you get messages for models with {#stable states} != 1
  models_stable_states = emba::get_stable_state_from_models_dir(models_dir)
  
  # same order => same model per row (also pruned if != 1 stable states)
  model_predictions = model_predictions[rownames(models_stable_states),]
  
  # calculate models MCC scores
  models_mcc = calculate_models_mcc(
    observed.model.predictions = model_predictions %>% select(all_of(observed_synergies)),
    unobserved.model.predictions = model_predictions %>% select(-all_of(observed_synergies)),
    number.of.drug.comb.tested = ncol(model_predictions))
  
  # calculate models fitness to AGS steady state
  models_fit = apply(models_stable_states[, names(steady_state)], 1, 
    usefun::get_percentage_of_matches, steady_state)
  
  # check data consistency
  stopifnot(all(names(models_mcc) == names(models_fit)))
  
  
  # bind all to one (OneForAll)
  df = bind_cols(mcc = models_mcc, fitness = models_fit, type = rep(type, length(models_fit)))
  
  data_list[[index]] = df
  index = index + 1
}

res = bind_rows(data_list)

# choose one (depends on the `res_dirs`)
#res_link = res
res_topo = res
#res_link_and_topo = res

#saveRDS(res_link, file = "data/res_link.rds")
saveRDS(res_topo, file = "data/res_topo.rds")
#saveRDS(res_link_and_topo, file = "data/res_link_and_topo.rds")
```

## Link operator mutated models {-}


```r
res_link = readRDS(file = "data/res_link.rds")

mean_fit = res_link %>% group_by(type) %>% summarise(mean = mean(fitness))

ggplot() + 
  geom_density(data = res_link, aes(x=fitness, group=type, fill=type), alpha=0.5, adjust=3) + 
  geom_vline(data = mean_fit, aes(xintercept = mean, color=type), linetype="dashed") +
  xlab("Fitness Score") +
  ylab("Density") + 
  ggtitle("Fitness density profile for different training data schemes") +
  ggpubr::theme_pubr()
```

<img src="index_files/figure-html/figures-2-1.png" width="2100" style="display: block; margin: auto;" />


```r
ggboxplot(data = res_link, x = "type", y = "mcc", 
  color = "type", palette = "Set1", order = c("rev", "prolif", "ss"),
  bxp.errorbar = TRUE, bxp.errorbar.width = 0.2,
  title = "Performance (MCC) of different training data schemes",
  ylab = "MCC score", xlab = "Training Data Type") + 
  stat_compare_means(label.x = 2.5)
```

<img src="index_files/figure-html/figures-3-1.png" width="2100" style="display: block; margin: auto;" />

```r
ggdensity(data = res_link, x = "mcc", color = "type", add = "mean",
  fill = "type", palette = "Set1", xlab = "MCC score",
  title = "MCC density profile for different training data schemes")
```

<img src="index_files/figure-html/figures-3-2.png" width="2100" style="display: block; margin: auto;" />

## Topology-mutated models {-}


```r
res_topo = readRDS(file = "data/res_topo.rds")

mean_fit = res_topo %>% group_by(type) %>% summarise(mean = mean(fitness))

ggplot() + 
  geom_density(data = res_topo, aes(x=fitness, group=type, fill=type), alpha=0.5, adjust=3) + 
  geom_vline(data = mean_fit, aes(xintercept = mean, color=type), linetype="dashed") +
  xlab("Fitness Score") +
  ylab("Density") + 
  ggtitle("Fitness density profile for different training data schemes") +
  ggpubr::theme_pubr()
```

<img src="index_files/figure-html/figures-4-1.png" width="2100" />


```r
ggboxplot(data = res_topo, x = "type", y = "mcc", 
  color = "type", palette = "Set1", order = c("rev", "prolif", "ss"),
  bxp.errorbar = TRUE, bxp.errorbar.width = 0.2,
  title = "Performance (MCC) of different training data schemes",
  ylab = "MCC score", xlab = "Training Data Type") + 
  stat_compare_means(label.x = 2.5)
```

<img src="index_files/figure-html/figures-5-1.png" width="2100" style="display: block; margin: auto;" />

```r
ggdensity(data = res_topo, x = "mcc", color = "type", add = "mean",
  fill = "type", palette = "Set1", xlab = "MCC score",
  title = "MCC density profile for different training data schemes")
```

<img src="index_files/figure-html/figures-5-2.png" width="2100" style="display: block; margin: auto;" />

## Both mutations on models {-}


```r
res_link_and_topo = readRDS(file = "data/res_link_and_topo.rds")

mean_fit = res_link_and_topo %>% group_by(type) %>% summarise(mean = mean(fitness))

ggplot() + 
  geom_density(data = res_link_and_topo, aes(x=fitness, group=type, fill=type), alpha=0.5, adjust=3) + 
  geom_vline(data = mean_fit, aes(xintercept = mean, color=type), linetype="dashed") +
  xlab("Fitness Score") +
  ylab("Density") + 
  ggtitle("Fitness density profile for different training data schemes") +
  ggpubr::theme_pubr()
```

<img src="index_files/figure-html/figures-6-1.png" width="2100" style="display: block; margin: auto;" />


```r
ggboxplot(data = res_link_and_topo, x = "type", y = "mcc", 
  color = "type", palette = "Set1", order = c("rev", "prolif", "ss"),
  bxp.errorbar = TRUE, bxp.errorbar.width = 0.2,
  title = "Performance (MCC) of different training data schemes",
  ylab = "MCC score", xlab = "Training Data Type") + 
  stat_compare_means(label.x = 2.5)
```

<img src="index_files/figure-html/figures-7-1.png" width="2100" style="display: block; margin: auto;" />

```r
ggdensity(data = res_link_and_topo, x = "mcc", color = "type", add = "mean",
  fill = "type", palette = "Set1", xlab = "MCC score",
  title = "MCC density profile for different training data schemes")
```

<img src="index_files/figure-html/figures-7-2.png" width="2100" style="display: block; margin: auto;" />

:::{.blue-box}
- Note that most of the models trained to the complete **reverse** steady state have an MCC score exactly equal to $0$, since the denominator is also zero ($TP+FP = 0$).
So, in a way the steady state models are better in that regard (i.e. they make predictions, even though most of them have negative MCC scores).
:::


# Fitness vs Ensemble performance {-}

:::{.green-box}
Is there any correlation between the **calibrated models fitness to the AGS steady state** and their ROC/PR AUC performance when they are **normalized to a random proliferative model ensemble**?
:::

In this section we use the same technique as in the [Flip training data analysis] section: we had already generated the flipped training data files which we now use again with the script [run_druglogics_synergy_training.sh](https://raw.githubusercontent.com/druglogics/gitsbe-model-analysis/master/cascade/fit-vs-performance-ags/data/run_druglogics_synergy_training.sh) from the `druglogics-synergy` repository root (version `1.2.0`), to get the simulation results for each of these training data files.

Differences are that now in the in the CASCADE 2.0 configuration file (`config`) we changed the number of simulations to $20$ for each training data file, the attractor tool used was `biolqm_stable_states` and the `synergy_method: bliss`.

Also, we used the `run_druglogics_synergy.sh` script at the root of the `druglogics-synergy` (script config: {`2.0`, `rand`, `150`, `biolqm_stable_states`, `bliss`}) repo to get the ensemble results of the **random (proliferative)** models that we will normalize the calibrated model performance at.

:::{.orange-box}
The dataset with the simulation results (`fit-vs-performance-results-bliss.tar.gz` file) is on [![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.3972340.svg)](https://doi.org/10.5281/zenodo.3972340).
:::

With the code below you can load this data (I have stored it in a `tibble` for convenience):

```r
# random proliferative results
random_ew_synergies_file = "data/cascade_2.0_rand_150sim_fixpoints_bliss_ensemblewise_synergies.tab"
random_ew_synergies_150sim = emba::get_synergy_scores(random_ew_synergies_file)

# 1 (positive/observed synergy) or 0 (negative/not observed) for all tested drug combinations
observed = sapply(random_ew_synergies_150sim$perturbation %in% observed_synergies, as.integer)

# get flipped training data results
data_dir = "/home/john/tmp/ags_paper_res/fit-vs-performance-results-bliss"

# define `beta` values for normalization
beta1 = -1
beta2 = -1.6

data_list = list()
index = 1
for (res_dir in list.dirs(data_dir, recursive = FALSE)) {
  ew_synergies_file = list.files(path = res_dir, pattern = "ensemblewise_synergies", full.names = TRUE)
  ew_ss_scores = emba::get_synergy_scores(ew_synergies_file)
  
  # get the models stable states
  models_dir = paste0(res_dir, "/models")
  # you get messages for models with {#stable states} != 1
  models_stable_states = emba::get_stable_state_from_models_dir(models_dir)
  
  # calculate models fitness to AGS steady state
  models_fit = apply(models_stable_states[, names(steady_state)], 1, 
   usefun::get_percentage_of_matches, steady_state)
  
  # calculate normalized model performance (ROC-AUC)
  pred = dplyr::bind_cols(random = random_ew_synergies_150sim %>% rename(random_score = score), 
    ss = ew_ss_scores %>% select(score) %>% rename(ss_score = score), 
    as_tibble_col(observed, column_name = "observed"))
  
  pred = pred %>% 
    mutate(combined_score1 = ss_score + beta1 * random_score) %>% 
    mutate(combined_score2 = ss_score + beta2 * random_score)
  
  res_roc1 = PRROC::roc.curve(scores.class0 = pred %>% pull(combined_score1) %>% (function(x) {-x}), 
    weights.class0 = pred %>% pull(observed))
  res_pr1 = PRROC::pr.curve(scores.class0 = pred %>% pull(combined_score1) %>% (function(x) {-x}), 
    weights.class0 = pred %>% pull(observed))
  res_roc2 = PRROC::roc.curve(scores.class0 = pred %>% pull(combined_score2) %>% (function(x) {-x}), 
    weights.class0 = pred %>% pull(observed))
  res_pr2 = PRROC::pr.curve(scores.class0 = pred %>% pull(combined_score2) %>% (function(x) {-x}), 
    weights.class0 = pred %>% pull(observed))
  
  # bind all to one (OneForAll)
  df = dplyr::bind_cols(roc_auc1 = res_roc1$auc, pr_auc1 = res_pr1$auc.davis.goadrich,
    roc_auc2 = res_roc2$auc, pr_auc2 = res_pr2$auc.davis.goadrich,
    avg_fit = mean(models_fit), median_fit = median(models_fit))
  data_list[[index]] = df
  index = index + 1
}

res = bind_rows(data_list)
saveRDS(res, file = "data/res_fit_aucs.rds")
```


```r
res = readRDS(file = "data/res_fit_aucs.rds")
```

We have used both a $\beta=-1$ normalization ($calibrated-random$) and a $\beta=-1.6$ ($calibrated-1.6\times random$, as candidate values that maximized the ROC and PR AUC based on another analysis we done prior to this one.
We quickly check for **correlation between the AUC values** produced with the **different beta parameters**:

```r
# ROC AUCs
cor(res$roc_auc1, res$roc_auc2)
```

```
[1] 0.8069361
```

```r
# PR AUCs
cor(res$pr_auc1, res$pr_auc2)
```

```
[1] 0.8479174
```


Some scatter plots to get a first idea (every point is one of the $205$ simulations, normalized to the random proliferative models):

```r
# beta = -1
ggscatter(data = res, x = "avg_fit", y = "roc_auc1", xlab = "Average Fitness",
  ylab = "ROC AUC", add = "reg.line", conf.int = TRUE, title = "β = -1",
  cor.coef = TRUE, cor.coeff.args = list(method = "kendall")) +
  theme(plot.title = element_text(hjust = 0.5))
```

<img src="index_files/figure-html/figures-8-1.png" width="2100" style="display: block; margin: auto;" />

```r
ggscatter(data = res, x = "avg_fit", y = "pr_auc1", xlab = "Average Fitness", 
  ylab = "PR AUC", add = "reg.line", conf.int = TRUE, title = "β = -1",
  cor.coef = TRUE, cor.coeff.args = list(method = "kendall")) +
  theme(plot.title = element_text(hjust = 0.5))
```

<img src="index_files/figure-html/figures-8-2.png" width="2100" style="display: block; margin: auto;" />

```r
# best beta (-1.6)
ggscatter(data = res, x = "avg_fit", y = "roc_auc2", xlab = "Average Fitness",
  ylab = "ROC AUC", add = "reg.line", conf.int = TRUE, title = "β = -1.6",
  cor.coef = TRUE, cor.coeff.args = list(method = "kendall")) +
  theme(plot.title = element_text(hjust = 0.5))
```

<img src="index_files/figure-html/figures-8-3.png" width="2100" style="display: block; margin: auto;" />

```r
ggscatter(data = res, x = "avg_fit", y = "pr_auc2", xlab = "Average Fitness", 
  ylab = "PR AUC", add = "reg.line", conf.int = TRUE, title = "β = -1.6",
  cor.coef = TRUE, cor.coeff.args = list(method = "kendall")) +
  theme(plot.title = element_text(hjust = 0.5))
```

<img src="index_files/figure-html/figures-8-4.png" width="2100" style="display: block; margin: auto;" />

```r
# ggscatter(data = res, x = "median_fit", y = "roc_auc")
# ggscatter(data = res, x = "median_fit", y = "pr_auc")
```

We will split the points to $4$ *fitness* groups (higher number group means **higher fitness**):


```r
fit_classes_num = 4
avg_fit_res = Ckmeans.1d.dp(x = res$avg_fit, k = fit_classes_num)
median_fit_res = Ckmeans.1d.dp(x = res$median_fit, k = fit_classes_num)

res = res %>% 
  add_column(avg_fit_group = avg_fit_res$cluster) %>% 
  add_column(median_fit_group = median_fit_res$cluster) 
```

And thus, we can show the following boxplots:


```r
my_comparisons = list(c(1,2), c(1,3), c(1,4))

# beta = -1
ggboxplot(data = res, x = "avg_fit_group", y = "roc_auc1", fill = "avg_fit_group",
  title = "β = -1", xlab = "Fitness Group (Average fitness)", ylab = "ROC AUC") + 
  stat_compare_means(comparisons = my_comparisons, method = "wilcox.test", label = "p.signif") +
  theme(plot.title = element_text(hjust = 0.5))
```

<img src="index_files/figure-html/figures-9-1.png" width="2100" style="display: block; margin: auto;" />

```r
ggboxplot(data = res, x = "avg_fit_group", y = "pr_auc1", fill = "avg_fit_group",
  title = "β = -1", xlab = "Fitness Group (Average fitness)", ylab = "PR AUC") + 
  stat_compare_means(comparisons = my_comparisons, method = "wilcox.test", label = "p.signif") +
  theme(plot.title = element_text(hjust = 0.5))
```

<img src="index_files/figure-html/figures-9-2.png" width="2100" style="display: block; margin: auto;" />

```r
# best beta (-1.6)
ggboxplot(data = res, x = "avg_fit_group", y = "roc_auc2", fill = "avg_fit_group",
  title = "β = -1.6", xlab = "Fitness Group (Average fitness)", ylab = "ROC AUC") + 
  stat_compare_means(comparisons = my_comparisons, method = "wilcox.test", label = "p.signif") +
  theme(plot.title = element_text(hjust = 0.5))
```

<img src="index_files/figure-html/figures-9-3.png" width="2100" style="display: block; margin: auto;" />

```r
ggboxplot(data = res, x = "avg_fit_group", y = "pr_auc2", fill = "avg_fit_group",
  title = "β = -1.6", xlab = "Fitness Group (Average fitness)", ylab = "PR AUC") + 
  stat_compare_means(comparisons = my_comparisons, method = "wilcox.test", label = "p.signif") +
  theme(plot.title = element_text(hjust = 0.5))
```

<img src="index_files/figure-html/figures-9-4.png" width="2100" style="display: block; margin: auto;" />

:::{.green-box}
- There is seems to be some correlation between **fitness group and AUC performance of the calibrated models normalized to the random proliferative** ones.
This manifests in **better PR AUC** for the higher fitness calibrated models subject to normalization. 
Note also that PR AUC is a better indicator of performance for our imbalanced dataset than the ROC AUC.
- Comparing the two normalization cases ($\beta=-1$ and $\beta=-1.6$) we observe that the results were in general better for $\beta=-1.6$ case (as was expected) but almost the same statistical differences and data *trends* were observed between the different fitness groups in each case.
:::



# R session info {-}


```r
xfun::session_info()
```

```
R version 3.6.3 (2020-02-29)
Platform: x86_64-pc-linux-gnu (64-bit)
Running under: Ubuntu 20.04.1 LTS

Locale:
  LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C              
  LC_TIME=en_US.UTF-8        LC_COLLATE=en_US.UTF-8    
  LC_MONETARY=en_US.UTF-8    LC_MESSAGES=en_US.UTF-8   
  LC_PAPER=en_US.UTF-8       LC_NAME=C                 
  LC_ADDRESS=C               LC_TELEPHONE=C            
  LC_MEASUREMENT=en_US.UTF-8 LC_IDENTIFICATION=C       

Package version:
  abind_1.4-5              assertthat_0.2.1         backports_1.1.10        
  base64enc_0.1.3          BH_1.72.0.3              bookdown_0.21           
  boot_1.3.25              broom_0.7.2              callr_3.5.1             
  car_3.0-10               carData_3.0-4            cellranger_1.1.0        
  Ckmeans.1d.dp_4.3.3      cli_2.1.0                clipr_0.7.1             
  colorspace_1.4-1         compiler_3.6.3           conquer_1.0.2           
  corrplot_0.84            cowplot_1.1.0            cpp11_0.2.3             
  crayon_1.3.4             curl_4.3                 data.table_1.13.2       
  desc_1.2.0               digest_0.6.27            dplyr_1.0.2             
  ellipsis_0.3.1           emba_0.1.8               evaluate_0.14           
  fansi_0.4.1              farver_2.0.3             forcats_0.5.0           
  foreign_0.8-75           gbRd_0.4-11              generics_0.0.2          
  ggplot2_3.3.2            ggpubr_0.4.0             ggrepel_0.8.2           
  ggsci_2.9                ggsignif_0.6.0           glue_1.4.2              
  graphics_3.6.3           grDevices_3.6.3          grid_3.6.3              
  gridExtra_2.3            gtable_0.3.0             haven_2.3.1             
  highr_0.8                hms_0.5.3                htmltools_0.5.0         
  htmlwidgets_1.5.2        igraph_1.2.6             isoband_0.2.2           
  jsonlite_1.7.1           knitr_1.30               labeling_0.4.2          
  lattice_0.20.41          lifecycle_0.2.0          lme4_1.1.25             
  magrittr_1.5             maptools_1.0.2           markdown_1.1            
  MASS_7.3.53              Matrix_1.2.18            MatrixModels_0.4.1      
  matrixStats_0.57.0       methods_3.6.3            mgcv_1.8.33             
  mime_0.9                 minqa_1.2.4              munsell_0.5.0           
  nlme_3.1.149             nloptr_1.2.2.2           nnet_7.3.14             
  openxlsx_4.2.2           parallel_3.6.3           pbkrtest_0.4.8.6        
  pillar_1.4.6             pkgbuild_1.1.0           pkgconfig_2.0.3         
  pkgload_1.1.0            polynom_1.4.0            praise_1.0.0            
  prettyunits_1.1.1        processx_3.4.4           progress_1.2.2          
  PRROC_1.3.1              ps_1.4.0                 purrr_0.3.4             
  quantreg_5.74            R6_2.4.1                 rbibutils_1.3           
  RColorBrewer_1.1.2       Rcpp_1.0.5               RcppArmadillo_0.10.1.0.0
  RcppEigen_0.3.3.7.0      Rdpack_2.0               readr_1.4.0             
  readxl_1.3.1             rematch_1.0.1            rio_0.5.16              
  rje_1.10.16              rlang_0.4.8              rmarkdown_2.5           
  rprojroot_1.3.2          rstatix_0.6.0            rstudioapi_0.11         
  scales_1.1.1             sp_1.4.4                 SparseM_1.78            
  splines_3.6.3            statmod_1.4.35           stats_3.6.3             
  stringi_1.5.3            stringr_1.4.0            testthat_2.3.2          
  tibble_3.0.4             tidyr_1.1.2              tidyselect_1.1.0        
  tinytex_0.26             tools_3.6.3              usefun_0.4.8            
  utf8_1.1.4               utils_3.6.3              vctrs_0.3.4             
  viridisLite_0.3.0        visNetwork_2.0.9         withr_2.3.0             
  xfun_0.18                xml2_1.3.2               yaml_2.2.1              
  zip_2.1.1               
```
