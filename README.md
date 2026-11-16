C Code Vulnerability Detection with Classical Machine Learning

Detecting vulnerable C functions with **Naive Bayes** and **linear SVM** on TF-IDF features, evaluated on the Devign dataset. The project also tests how well the models generalize across codebases, and the answer is: not well. That finding is the most interesting part of the work.

TL;DR

- Task: binary classification of C functions (vulnerable vs. safe).
- Best test results: about **62% accuracy** (vs. 54.4% for always guessing "safe"), with vulnerable-class F1 of about 0.58 to 0.59.
- Naive Bayes and SVM perform about the same once both are tuned.
- **Cross-project test: accuracy drops to chance (about 50%)**, which suggests the models mostly learn project-specific vocabulary rather than general vulnerability patterns.

Dataset

[Devign](https://sites.google.com/view/devign) (Zhou et al., 2019): 27,318 manually labeled C functions from two open-source projects, QEMU (17,549) and FFmpeg (9,769). Each function is labeled vulnerable (1) or non-vulnerable (0). Class balance: 54.4% safe, 45.6% vulnerable.

The dataset is not included in this repository. Download `function.json` from the Devign website and place it in `data/`.

Split used: 80/10/10 train/validation/test, stratified on the label, `random_state=42` (21,854 / 2,732 / 2,732 functions).

 Method

1. **Cleaning:** remove block and line comments, collapse whitespace.
2. **Features:** TF-IDF over code-aware tokens (identifiers, numbers and operators such as `->`, `&&`, `==` kept as separate tokens), unigrams and bigrams, `min_df=2`, top 50,000 features. The vectorizer is fit on the training set only.
3. **Models:**
   - Multinomial Naive Bayes, `alpha=0.01`
   - Linear SVM (`LinearSVC`), `C=0.5`, `class_weight="balanced"`
   - Dummy baseline: always predicts the majority class
4. **Tuning:** hyperparameters were chosen on the validation set only. The test set was evaluated once, after the settings were frozen.

Results

Test set (2,732 functions):

| Model | Accuracy | Precision (vuln) | Recall (vuln) | F1 (vuln) |
|---|---|---|---|---|
| Dummy (always "safe") | 0.544 | n/a | 0.000 | n/a |
| Naive Bayes | 0.625 | 0.592 | 0.573 | 0.582 |
| Linear SVM (balanced) | 0.616 | 0.575 | 0.608 | 0.591 |

Notes:

- The two models are statistically indistinguishable. With 2,732 test samples, the margin of error on accuracy is roughly +/-1.8 points.
- The balanced SVM catches more vulnerable functions (higher recall) at the cost of more false alarms (lower precision). For a security tool, that trade-off can be preferable.
- Validation scores were slightly higher than test scores (SVM: 0.636 vs 0.616). Tuning on the validation set makes it a mildly optimistic estimate, so the test numbers are the ones to trust.

 What are the models actually learning?

The highest-weighted SVM features show that much of the signal is not about vulnerabilities:

- Tokens such as `target_phys_addr_t` (pushes toward safe) and `hwaddr` (pushes toward vulnerable) look like two eras of the same QEMU codebase, so the model partly encodes *when* a function was written.
- Subsystem-specific names such as `qdev_init`, `ahci`, `config_reg` and `gen_inval_exception` push toward vulnerable, which reflects how the dataset is labeled in those parts of the code, not a general vulnerability pattern.
- Structural tokens (`}`, `{`, `; }`) appear near the top, probably acting as proxies for function length and shape.
- Classic danger signs such as `memcpy` or `strcpy` do not appear among the top 20 features in either direction.

 Cross-project generalization

To test this directly, the models were trained on one project and tested on the other, and compared with within-project training. Settings are the same as above.

| Trained on -> tested on | Naive Bayes acc. | SVM acc. |
|---|---|---|
| QEMU -> FFmpeg | 0.496 | 0.514 |
| FFmpeg -> QEMU | 0.483 | 0.496 |
| QEMU -> QEMU (80/20 split) | 0.658 | 0.657 |
| FFmpeg -> FFmpeg (80/20 split) | 0.585 | 0.564 |

Within a project, the models beat the dummy baseline by roughly 7 to 8 points. Across projects, accuracy falls to about 50%, i.e. chance level. The class balance differs between the two projects, so the dummy baseline is not comparable across directions; a coin flip (0.50) is the fairer reference.

Conclusion: with bag-of-words features on this dataset, the models learn project-specific vocabulary and style, and the signal does not transfer to a new codebase.

 Limitations

- Bag-of-words cannot read code logic. Vulnerabilities such as missing bounds checks or unchecked sizes depend on how statements relate, which TF-IDF features cannot express.
- Label noise.** Devign's labels are derived from security-related commits, and prior work has reported a notable fraction of mislabeled samples, which limits any model's ceiling.
- Possible split leakage. Functions were split randomly, so a vulnerable function and its patched version (from the same commit) may fall on different sides of the split, which can inflate within-project scores. Splitting by `commit_id` would address this.
- Two projects, one language. Results may not hold for other codebases or languages.

Reproducing the results

```bash
git clone https://github.com/YOUR-USERNAME/vuln-detection-ml.git
cd vuln-detection-ml
python -m venv venv
venv\Scripts\activate        # Windows (Mac/Linux: source venv/bin/activate)
pip install -r requirements.txt
```

1. Download `function.json` from the Devign website into `data/`.
2. Run the notebooks in order:
   - `notebooks/01_explore.ipynb` explores the data and writes the train/validation/test split to `data/`.
   - `notebooks/02_baseline.ipynb` builds features, tunes and evaluates Naive Bayes and SVM.
   - `notebooks/03_cross_project.ipynb` runs the cross-project experiment.

Project structure

```
vuln-detection-ml/
├── data/            # datasets (not committed)
├── notebooks/       # 01_explore, 02_baseline, 03_cross_project
├── src/             # reusable code
├── results/         # plots and metrics
├── requirements.txt
└── README.md
```

Tech stack

Python, scikit-learn, pandas, NumPy, Jupyter.

Reference

Zhou, Y., Liu, S., Siow, J., Du, X., Liu, Y. *Devign: Effective Vulnerability Identification by Learning Comprehensive Program Semantics via Graph Neural Networks.* NeurIPS 2019.
