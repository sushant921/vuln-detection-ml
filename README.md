# Code Vulnerability Detection: Classical ML vs Neural Networks

Can machine learning tell vulnerable C functions from safe ones? This project compares **classical ML** (Naive Bayes, linear SVM) with **neural networks** (an MLP on TF-IDF features and a fine-tuned **CodeBERT**) on the Devign dataset, and tests how well each one carries over to a codebase it has not seen.

## TL;DR

- Task: binary classification of C functions (vulnerable vs. safe).
- On the held-out test set, all models land in a narrow band: about **60 to 64% accuracy** (always guessing "safe" gives 54.4%).
- The fine-tuned **CodeBERT is the best model** (64.2% accuracy, 0.627 F1 on the vulnerable class), but only by about 2 to 4 points, which is close to the noise level of the test set.
- Adding a neural network on the same bag-of-words features (MLP) gave **no improvement** over Naive Bayes and SVM.
- **Cross-project test: every model, including CodeBERT, drops to about chance level** when trained on one project and tested on the other. The scores are mostly driven by project-specific vocabulary and style, not by general vulnerability patterns.

## Dataset

[Devign](https://sites.google.com/view/devign) (Zhou et al., 2019): 27,318 manually labeled C functions from two open-source projects, QEMU (17,549) and FFmpeg (9,769). Each function is labeled vulnerable (1) or non-vulnerable (0). Class balance: 54.4% safe, 45.6% vulnerable.

The dataset is not included in this repository. Download `function.json` from the Devign website and place it in `data/`.

Split: 80/10/10 train/validation/test, stratified on the label, `random_state=42` (21,854 / 2,732 / 2,732 functions). All models use the same split.

## Methods

**Preprocessing (all models):** remove block and line comments, collapse whitespace.

**Classical ML**
- Features: TF-IDF over code-aware tokens (identifiers, numbers and operators such as `->`, `&&`, `==` kept as separate tokens), unigrams and bigrams, `min_df=2`, top 50,000 features, fit on the training set only.
- Multinomial Naive Bayes, `alpha=0.01`.
- Linear SVM (`LinearSVC`), `C=0.5`, `class_weight="balanced"`.

**Neural networks**
- **MLP on TF-IDF:** the same 50,000 features feed a 256-unit hidden layer (ReLU, dropout 0.5) and a 2-class output. Adam (lr 1e-3), class-weighted loss, 10 epochs, with the best epoch chosen on validation F1.
- **CodeBERT:** `microsoft/codebert-base` fine-tuned end to end with a classification head. AdamW (lr 2e-5), linear warmup and decay, class-weighted loss, 3 epochs, mixed precision (fp16), batch size 8 with 2 accumulation steps, inputs truncated to 512 tokens. Training on the Devign training set took about 20 minutes on an RTX 2070 Super (8 GB).

**Baseline:** a dummy classifier that always predicts the majority class.

**Evaluation protocol:** hyperparameters were chosen on the validation set only. Each model was evaluated on the test set once, after its settings were frozen. CodeBERT was trained twice (the first run's weights were lost in a notebook restart before testing), and only the second run was evaluated on the test set.

## Results

Test set (2,732 functions):

| Model | Accuracy | Precision (vuln) | Recall (vuln) | F1 (vuln) |
|---|---|---|---|---|
| Dummy (always "safe") | 0.544 | n/a | 0.000 | n/a |
| Naive Bayes | 0.625 | 0.592 | 0.573 | 0.582 |
| Linear SVM (balanced) | 0.616 | 0.575 | 0.608 | 0.591 |
| MLP on TF-IDF | 0.601 | 0.555 | 0.632 | 0.591 |
| **CodeBERT (fine-tuned)** | **0.642** | 0.597 | **0.661** | **0.627** |

How to read this:

- With 2,732 test samples, the margin of error on accuracy is roughly +/-1.8 points. CodeBERT's lead (about 2 to 4 points on accuracy, 4 to 5 on F1) is suggestive, not conclusive.
- Run-to-run variation is real: two CodeBERT training runs differed by about 1 point on validation scores.
- The MLP is no better than the simpler models. The bottleneck is the bag-of-words features, not the model's capacity.
- CodeBERT catches more vulnerable functions (recall 0.661) but also flags more safe functions as vulnerable (555 of 1,486 in the test set).

## Does it generalize to a new codebase?

Each model was trained on one project and tested on the other, and compared with training and testing within the same project (80/20 split per project). Settings are the same as above. The "trivial" column is the better of always-safe and always-vulnerable on that test set, so a score is only meaningful if it clearly beats it.

Accuracy:

| Trained on -> tested on | Trivial | Naive Bayes | SVM | CodeBERT |
|---|---|---|---|---|
| QEMU -> FFmpeg | 0.510 | 0.496 | 0.514 | 0.528 |
| FFmpeg -> QEMU | 0.574 | 0.483 | 0.496 | 0.557 |
| QEMU -> QEMU | 0.574 | 0.658 | 0.657 | 0.666 |
| FFmpeg -> FFmpeg | 0.510 | 0.585 | 0.564 | 0.615 |

Findings:

- **Within a project**, every model beats the trivial baseline by roughly 7 to 10 points, and CodeBERT is the best or tied-best on accuracy.
- **Across projects**, all models fall to about chance. CodeBERT is slightly less bad than the TF-IDF models, but FFmpeg -> QEMU (0.557) is still below the 0.574 you would get by always guessing "safe", and CodeBERT's recall on the vulnerable class is only about 0.33 to 0.36 in both directions: on an unfamiliar codebase it mostly says "safe".
- Pretraining on code therefore gives a modest boost within a project but does not produce a detector that works on new code.

## What are the classical models learning?

The highest-weighted SVM features show that much of the signal is not about vulnerabilities:

- `target_phys_addr_t` (pushes toward safe) and `hwaddr` (pushes toward vulnerable) look like two eras of the same QEMU codebase, so the model partly encodes *when* a function was written.
- Subsystem-specific names such as `qdev_init`, `ahci`, `config_reg` and `gen_inval_exception` push toward vulnerable, which reflects how the dataset is labeled in those parts of the code, not a general vulnerability pattern.
- Structural tokens (`}`, `{`, `; }`) appear near the top, probably acting as proxies for function length and shape.
- Classic danger signs such as `memcpy` or `strcpy` do not appear among the top 20 features in either direction.

## Limitations

- **Bag-of-words cannot read code logic.** Vulnerabilities such as missing bounds checks depend on how statements relate, which TF-IDF features cannot express.
- **CodeBERT constraints.** Inputs are truncated at 512 tokens, so long functions lose their tails. CodeBERT's pretraining covered Python, Java, JavaScript, PHP, Ruby and Go, not C.
- **Label noise.** Devign's labels come from security-related commits, and prior work has reported a notable fraction of mislabeled samples, which limits any model's ceiling.
- **Possible split leakage.** Functions were split randomly, so a vulnerable function and its patched version (from the same commit) may fall on different sides of the split, which can inflate within-project scores.
- **Single runs.** Each number comes from one training run. Differences of a point or two should not be over-interpreted.
- **Two projects, one language.** Results may not hold for other codebases or languages.

## Future work

- Group-aware split by `commit_id` to remove possible leakage.
- Statistical comparison of models on the same test samples (for example McNemar's test) and repeated runs with different seeds.
- Handle long functions (sliding windows or chunking) and try code models pretrained on C/C++.
- Extend to C++ (for example DiverseVul) and PHP (for example CVEfixes), and evaluate on additional projects.

## Reproducing the results

```bash
git clone https://github.com/sushant921/vuln-detection-ml.git
cd vuln-detection-ml
python -m venv venv
venv\Scripts\activate        # Windows (Mac/Linux: source venv/bin/activate)
pip install -r requirements.txt
```

Install PyTorch first using the command from [pytorch.org](https://pytorch.org) that matches your hardware (the pinned CUDA build in `requirements.txt` will not install from the default package index). A CUDA GPU is needed for the CodeBERT notebooks; the rest runs on CPU.

1. Download `function.json` from the Devign website into `data/`.
2. Run the notebooks in order:
   - `notebooks/01_explore.ipynb`: data exploration and the train/validation/test split.
   - `notebooks/02_baseline.ipynb`: TF-IDF features, Naive Bayes and SVM, tuning, token analysis, test evaluation.
   - `notebooks/03_cross_project.ipynb`: cross-project experiment for Naive Bayes and SVM.
   - `notebooks/04_neural_net.ipynb`: MLP on TF-IDF features.
   - `notebooks/05_codebert.ipynb`: CodeBERT fine-tuning and test evaluation.
   - `notebooks/06_codebert_cross_project.ipynb`: CodeBERT cross-project experiment (results saved to `results/codebert_cross_project.csv`).

## Project structure

```
vuln-detection-ml/
├── data/            # datasets (not committed)
├── models/          # saved model weights (not committed)
├── notebooks/       # 01_explore ... 06_codebert_cross_project
├── results/         # saved experiment results
├── src/             # reusable code
├── requirements.txt
└── README.md
```

## Tech stack

Python, scikit-learn, pandas, NumPy, PyTorch, Hugging Face Transformers, Jupyter.

## Reference

Zhou, Y., Liu, S., Siow, J., Du, X., Liu, Y. *Devign: Effective Vulnerability Identification by Learning Comprehensive Program Semantics via Graph Neural Networks.* NeurIPS 2019.

Feng, Z. et al. *CodeBERT: A Pre-Trained Model for Programming and Natural Languages.* Findings of EMNLP 2020.
