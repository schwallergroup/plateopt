# Parallel and Resource-Efficient Bayesian Reaction Optimization

📄 Paper: [![ChemRxiv](https://img.shields.io/badge/ChemRxiv-10.26434%2Fchemrxiv.15012572%2Fv1-007398.svg)](https://chemrxiv.org/doi/full/10.26434/chemrxiv.15012572/v1)

---

![Overview of workflow](assets/banner.png)

## 🔍 Overview

PlateOpt is a reaction optimization framework that designs entire high-throughput experimentation (HTE) well plates rather than individual reactions. Instead of selecting each well independently, it proposes structured grids (e.g., 8 ligands × 3 bases × 2 solvents) in which each reagent spans a full row, column or block. This means every plate can be prepared with multichannel pipettes or simple liquid handlers. A Gaussian process scores every condition in the search space, the scores are aggregated per reagent to fill each possible plate layout, and the highest-scoring plate within a stock-solution budget is run next.

- **Plate-level design.** All two- and three-axis layouts for 24-, 48- and 96-well plates are enumerated, with any component type on any axis (temperature is held fixed within each plate).
- **Practical by construction.** Every proposed plate stays within a stock-solution budget (30 by default), typically needing ~25 stock solutions per plate compared with 150–250 for unconstrained batch BO.
- **Improved optimization efficiency.** On Buchwald–Hartwig and sulfonamide N-arylation benchmarks, PlateOpt needs 5.7–6.3× fewer stock solutions per percentage point of yield gained than unconstrained batch BO.
- **Prospective validation.** In a Cu-catalyzed C–O coupling, six 48-condition plates (1.0% of a 28,800-condition space) raised the yield from 14.5% to 72%, and the optimized conditions gave 99% in batch.

## 📦 Installation

```bash
git clone https://github.com/schwallergroup/plateopt.git
cd plateopt
conda create --name plateopt python=3.8
conda activate plateopt
pip install -r requirements.txt
pip install -e .
```

Developed and tested on Linux with CUDA 11.7 GPUs. On Linux, `pip` installs PyTorch 2.0.1 built for CUDA 11.7; the code also runs on CPU (slower). To use a conda-managed CUDA toolkit instead, run this before the `pip` steps:

```bash
conda install pytorch==2.0.1 torchvision==0.15.2 torchaudio==2.0.2 pytorch-cuda=11.7 -c pytorch -c nvidia
```

## Repository structure

```
src/plateopt/         package: GP model, scoring, plate design, plate visualisation
scripts/              retrospective benchmark scripts
benchmark_datasets/   virtual benchmark datasets
notebooks/            tutorial: suggest_next_plate.ipynb
publication/          experimental HTE campaign from the publication
pytests/              unit tests
```

## Retrospective benchmarks

The scripts simulate optimization campaigns on the virtual benchmark datasets and compare PlateOpt with an unconstrained Bayesian optimization baseline. Run them from `scripts/`, since paths are relative to that folder:

```bash
cd scripts
mkdir -p ../benchmark_results/

# PlateOpt: one structured plate per iteration
python run_template.py --dataset buchwald --scoring_method qnei --metric mean --num_samples 48 --threshold 30 --seed 1

# baseline: unconstrained sequential qNEI batch per iteration
python run.py --dataset buchwald --num_samples 48 --seed 1
```

| Option | Scripts | Meaning |
|---|---|---|
| `--dataset` | all | `buchwald` or `sulfonamide` |
| `--init_size` | all | initial quasi-random sampled reactions |
| `--num_samples` | all | reactions per iteration (plate size for `run_template.py`) |
| `--n_iterations` | all | optimization iterations |
| `--seed` | all | random seed |
| `--scoring_method` | `run_template.py` | acquisition function per reaction: `qnei`, `ucb`, `qpo` |
| `--metric` | `run_template.py` | aggregation of scores per component, e.g. `mean`, `max`, `ucb` |
| `--threshold` | `run_template.py` | maximum stock solutions allowed per plate |

Each run appends one row to a CSV file:

| Script | Output |
|---|---|
| `run_template.py` | `benchmark_results/results_<dataset>_template.csv` |
| `run.py` | `benchmark_results/results_<dataset>_baseline.csv` |

The row records the settings, the best yield found up to each iteration (`Max Normalized Yield (%) Iteration i`) and the stock solutions needed per iteration (`Stock solutions per Iteration i`). `run_template.py` and `run.py` report yield as a percentage of the best ground truth yield in the dataset.

### Benchmark datasets

| File | Reactions | Components |
|---|---|---|
| `benchmark_datasets/buchwald_virtual_benchmark.csv` | 38,610 | 80 ligands, 15 solvents, 3 precursors, 11 bases |
| `benchmark_datasets/sulfonamide_virtual_benchmark.csv` | 21,529 | 82 ligands, 11 solvents, 3 precursors, 8 bases |

Columns: `rxn_id`, the component names, reaction descriptors normalised to [0, 1], and the reaction objectives `AP yield` and `AP selectivity`. Full details of the virtual benchmarks are provided in the manuscript.

## 🧪 Prospective campaigns

[`notebooks/suggest_next_plate.ipynb`](notebooks/suggest_next_plate.ipynb) walks through suggesting the next HTE plate from experimental results: fit the GP, score the chemical space, design the plate under a stock-solution budget, and plot the plate layout with well positions.

```bash
jupyter notebook notebooks/suggest_next_plate.ipynb
```

It uses the data in `publication/`. To run your own campaign, provide:

- **a chemical space CSV** listing every candidate reaction: a `rxn_id` column equal to the row position (0, 1, 2, ...), one column per component, then numerical descriptors normalised to [0, 1];
- **training data CSVs**: the reactions run so far with the same columns as the chemical space, plus a `Yield` column.

Two column names have special meaning:

- `temperature`: held at a single value on each plate and not counted as a stock solution. Name your temperature column exactly `temperature`; under any other name it is treated as an ordinary component and can vary within a plate.
- `solvent`: used to count stock solutions per plate, as (2 + number of non-solvent components) × number of solvents, where 2 is the two substrates. Name your solvent column exactly `solvent`; without it, plates are counted as needing 0 stock solutions and `stock_solution_threshold` has no effect.

Set `train_plates`, `plate_size` and `stock_solution_threshold` in the notebook. After running a suggested plate, add its results to the training data and run the notebook again.

## 🔬 Experimental data

The HTE experimental data reported in the manuscript can be found in `publication/`:

| File | Contents |
|---|---|
| `chemical_space.csv` | 28,800 candidate reactions (30 ligands × 8 solvents × 6 bases × 10 copper precursors × 2 temperatures) with their associated descriptors |
| `plate_<i>/plate_<i>_train_data.csv` | The 48 reactions run on plate *i* (1–6) with their associated descriptors and measured `Yield` (%) |
| `all_hte_reaction_data.csv` | All 288 reactions from plates 1–6: components and `Yield` (%) |

## Citation

If you use PlateOpt, please cite:

```bibtex
@article{katzenburg2026plateopt,
  title   = {Parallel and Resource-Efficient {Bayesian} Reaction Optimization},
  author  = {Katzenburg, Felix and Boser, Florian and Sin, Joshua W. and Rankovi{\'c}, Bojana and Kopf, Sarina and Schreiner, Florian and Coersmeier, Kea and Daniliuc, Constantin G. and Hansen, Michael Ryan and Schwaller, Philippe and Glorius, Frank},
  journal = {ChemRxiv},
  year    = {2026},
  doi     = {10.26434/chemrxiv.15012572/v1},
  url     = {https://chemrxiv.org/doi/full/10.26434/chemrxiv.15012572/v1}
}
```

## License

Released under the MIT License, see [LICENSE](LICENSE).

## Acknowledgements

This work was supported by the Deutsche Forschungsgemeinschaft (Priority Program "Molecular Machine Learning", SPP 2363), the European Research Council (Advanced Grant Agreement No. 101098156, HighEnT), Roche and its Technology Innovation and Science (TIS) initiative, and NCCR Catalysis (grant no. 225147), a National Centre of Competence in Research funded by the Swiss National Science Foundation.
