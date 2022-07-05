# Documentation for Workstream 3 assets

**Important notice**

**Workstream 3 was paused indefinitely during active development. Assets and results should therefore not be interpreted as final; nor should they be assumed to generalize to data/parameters outside of those used during development.**

## Jump to

- [File descriptions](#files)
- [Title 22 model information](#title22)
- [Warmstart algorithm information](#warmstart)
- [CPLEX parameter tuning information](#params)

## Summary {#summary}

Workstream 3 was a Spring 2022 partnership with IBM and focused on creating a decision optimization model for the ED service line that:

    1. provided schedules at 15-min intervals;
    2. was compliant with Title 22 regulations;
    3. provided a feasible solution in less that 4-hours with associated productive FTE no greater than 10% of that produced by a manually created schedule.

The model was developed using 2019 CLT Emergency Department data, primarily using Scenario 1 parameters (max shift starts: 8; min shift length: 10; max shift length: 12).

Workstream 3, as well as the assets contained in this repo, fall into four general buckets:

    1. updates to metrics to account for 15-min blocks
    2. updates to the ED model to account for Title 22 requirements
    3. development of warmstart algorithm(s) for improved performance
    4. CPLEX parameter tuning for improved performance

## Files {#files}

| Category | Filename | Description |
| -------- | -------- | ----------- |
| 15-min data | providence_metrics.py | adds new class for generating metrics at 15-min blocks [^1] |
| | ED_DATA_TRANSFORM_TITLE22.ipynb | sample usage of new class as described above |
| | TITLE22_VOLUME_QUALITY_CHECK.ipynb | validation for 15-min block metric updates |
| Title 22 model | setup_functions_title22.py | decision variable/constraints for Title 22 model |
| | ed_functions_title22.py | ED-specific constraints/role dict setup for Title 22 model |
| | JOB_NOTEBOOK_RUN_ED_MODEL.ipynb | notebook used in WS jobs to run experiments on Title 22 model [^1] |
| | title22_run_utils.py | utility functions to facilitate experiments on Title 22 model |
| | title22_unit_tests.py | functions for running tests on outputted schedules |
| | JOB_RESULT_ANALYSIS.ipynb | sample usage of unit tests and output summary methods |
| warmstart | warmstart_helpers.py | includes class for generating warmstart solution for Title 22 model [^1] |
| CPLEX parameters | current_best_cplex_params.pkl | dict of best 3 sets of CPLEX params |
| miscellaneous | providence_helpers.py | includes updates for saving/reading data and generalizes to other WS Projects |
[^1] See asset for more detailed information

## Title 22 model {#title22}

Compared to the original 1-hour decision optimization model for ED, the Title 22 model includes the following updates:

- range of indices accommodate 15-min block data--indices ranges from [0,95] for "short" variables and [0,143] for "long" variables
- inclusion of two types of breaks: 15-min (paid) breaks (appear as `B15`), and 30-min (unpaid) lunch breaks (appear as two consecutive `B`s)
- operational constraints for breaks, including: no `B15` in first/last shift hour; no `B` in first/last two shift hours; breaks only on 00 or 30 of the hour; at least one hour gap between breaks
- Title 22 constraints, which enforce the number of each type of break and their sequence as a function of a staff's shift length

### Title 22 break requirements included in the model

| shift length | no. of 15-min breaks | no. of 30-min breaks | sequencing |
| ------------ | -------------------- | -------------------- | ---------- |
| x <= 4 | 1 | 0 | n/a |
| 5 <= x <= 7 | 0 | 1 | n/a |
| 8 <= x <= 11 | 2 | 1 | `B15`-`BB`-`B15` |
| x >= 12 | 3 | 1 | `B15`-`BB`-`B15`-`B15` or `B15`-`B15`-`BB`-`B15` |

### 15-min/Title 22 constraints in the model



## Warmstart algorithm {#warmstart}

A warmstart is an initial feasible solution provided to the CPLEX model that helps the model determine the structure of the schedule to speed up the execution. Our goal was to create the as optimal a solution as we could heuristically.

Improvements in our current warmstart (compared to the 1 hour version):

1. Initial block with first shift start at the peak of the requirement.
2. Extending shift starts based on volume peaks and deciding the best one that
3. Iteratively adding shifts based on volume requirements but selecting breaks in the blocks that have minimum requirement.

The function also iterates over:

- Different number of shift starts
- Different algorithms (v4 , v9, v11)

To select the most optimum schedule for each role. The `warmstart_helpers.py` module contains the code and a summary of the changes between the different warmstart versions.

## CPLEX parameter tuning {#params}

The notebook `JOB_NOTEBOOK_RUN_ED_MODEL.ipynb` facilitates experimentation, including tuning of CPLEX parameters by taking any arbitrary subset as environment variables that are then passed to the `run_model_now()` function. Instructions for usage can be found within the notebook itself.

| CPLEX parameter[^1] | Run 48 | Run 53 | Run 58 |
| --------------- | ------ | ------ | ------ |
| parameters.mip.cuts.implied | 2 | 0[^2] | 0[^2] |
| parameters.mip.cuts.localimplied | 3 | 0[^2] | 0[^2] |
| parameters.mip.strategy.subalgorithm | 0[^2] | 0[^2] | 1 |
| parameters.mip.strategy.heuristifreq | 0[^2] | -1 | -1 |
| parameters.solutiontype | 2 | 2 | 2 |
| parameters.emphasis.mip | 1 | 1 | 1 |
| parameters.mip.tolerances.mipgap | 0.1 | 0.1 | 0.1 |
| parameters.mip.polishafter.mipgap | 0.2 | 0.2 | 0.2 |
| parameters.mip.strategy.startalgorithm | 4 | 4 | 4 |
| parameters.mip.strategy.rinsheur | 20 | 20 | 20 |
| parameters.mip.strategy.variableselect | 4 | 4 | 4 |
| parameters.threads | 8 | 8 | 8 |
| parameters.randomseed | 43 | 43 | 43 |
| parameters.timelimit | 14400 | 14400 | 14400 |
| parameters.display | 5 | 5 | 5 |
[^1] [CPLEX parameter documentation](https://www.ibm.com/docs/en/SSSA5P_12.8.0/ilog.odms.studio.help/pdf/paramcplex.pdf)
[^2] Default CPLEX value
