# expiMap: course project on scArches

Course project (Bioinformatics, Amirkabir University, 2024-25) built on
[scArches / expiMap](https://github.com/theislab/scarches) (Lotfollahi et al., *Nat. Biotechnol.* 2022).
The original project modified expiMap and judged the result by looking at UMAPs.
This repository holds the cleaned project files, together with scripts that measure the modifications with numbers.

## Changes to the model
The code lives in `src/scarches/`, a copy of scArches v0.6.1. Two changes were made to expiMap.

- **Extra encoder layer.** An additional `Linear + ReLU` layer is added to the encoder just before the mean and variance layers. It is controlled by the option `use_extra_layer`.
- **Correlation penalty.** A penalty on the latent variables is added to the loss. It is the sum of the squared off-diagonal entries of the latent covariance matrix, multiplied by the weight `alpha_corr` (1e-3). It is controlled by the option `use_corr_penalty`.

## Experiment
Each run uses the setup of the scArches expiMap tutorial. First, a reference model is trained on the PBMC data (4 studies, 32,484 cells) using Reactome pathways, 2000 highly variable genes, three hidden layers of 256 nodes, the negative binomial loss and a group lasso weight of 0.7. Training lasts up to 400 epochs with early stopping. Then the Kang data (13,576 cells, control and interferon-stimulated) is mapped onto the reference with architecture surgery, which keeps the reference model frozen and trains only the new input weights for the Kang study.

This is done for four versions of the model: the unmodified baseline, the extra layer alone, the correlation penalty alone, and both changes together. Each version is run with three random seeds (0, 1 and 2), and the results are reported as the mean and standard deviation over the seeds.

The results are measured in four ways. Batch mixing is measured with iLISI and batch ASW, and cell-type separation with cell-type ASW, NMI and ARI. Label transfer is measured by the accuracy and macro F1 of predicting the Kang cell types from the reference. Finally, three pathway checks look at how correlated the pathway latents are, how much the reference cells move after surgery, and how well the interferon signalling latent separates stimulated from control cells. All numbers are saved in `expimap_results/results_summary.csv` (mean ± std over the seeds) and `expimap_results/results.csv` (every run).

## Results
All 12 runs finished on a Colab T4 GPU, and each took between 10 and 15 minutes. The numbers below are means over the three seeds, with the standard deviation in brackets where it matters. Label transfer is measured on the 91.9% of Kang cells whose cell type also exists in the reference.

| Version | Query iLISI | Query batch ASW | Cell-type ASW | NMI | Label-transfer accuracy | Latent correlation | Interferon AUROC |
|---|---|---|---|---|---|---|---|
| Baseline | 0.135 ± 0.007 | 0.903 ± 0.002 | 0.549 ± 0.001 | 0.696 ± 0.004 | 0.855 ± 0.012 | 0.235 ± 0.015 | 0.985 ± 0.007 |
| Extra layer | 0.149 ± 0.004 | 0.893 ± 0.005 | 0.552 ± 0.000 | 0.710 ± 0.005 | 0.836 ± 0.007 | 0.102 ± 0.022 | 0.994 ± 0.001 |
| Correlation penalty | 0.145 ± 0.002 | 0.899 ± 0.003 | 0.549 ± 0.002 | 0.698 ± 0.004 | 0.832 ± 0.013 | 0.233 ± 0.003 | 0.991 ± 0.005 |
| Both | 0.158 ± 0.010 | 0.894 ± 0.004 | 0.547 ± 0.001 | 0.683 ± 0.006 | 0.757 ± 0.018 | 0.105 ± 0.011 | 0.535 ± 0.057 |

Query iLISI and query batch ASW are measured on the joint reference and Kang embedding. Cell-type ASW and NMI are measured on the reference. Latent correlation is the mean absolute off-diagonal correlation between pathway latents.

The extra encoder layer and the correlation penalty on their own behave much like the baseline for most measures. Cell-type separation is almost the same in all four versions, with a reference cell-type ASW of about 0.55 and an NMI between 0.68 and 0.71. Batch mixing of the Kang query is slightly higher with the changes (joint iLISI 0.135 for the baseline and 0.158 for both changes together), while batch ASW is slightly lower (0.903 for the baseline and 0.894 for both).

The extra layer is the only change that clearly alters the pathway latents. The mean absolute off-diagonal correlation between them drops from 0.235 for the baseline to 0.102 with the extra layer, while the correlation penalty alone leaves it at 0.233. In every version all pathway terms stay active after surgery and the reference cells do not move (r = 1.0).

Using both changes together has the largest effect. Label-transfer accuracy falls from 0.855 (0.012) for the baseline to 0.757 (0.018), and the interferon signalling AUROC falls from 0.985 to 0.535 (0.057), close to chance level, so that node no longer separates stimulated from control cells. The extra layer alone (accuracy 0.836, AUROC 0.994) and the penalty alone (accuracy 0.832, AUROC 0.991) do not show this drop.

## Run
```
make env            # local venv from requirements.lock (arm64 macOS, Python 3.12)
python download_data.py
make smoke          # 3-epoch CPU test, a few minutes
make run            # full experiment (slow on CPU; use colab_run.ipynb with a GPU)
```
