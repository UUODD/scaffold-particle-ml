# Scaffold Particle Simulation and Machine Learning

Python tools and datasets for particle packing simulation, contact-chain analysis, simulation geometry export, and machine-learning-assisted scaffold formulation selection.

The repository provides two standalone graphical applications:

- **Particle random distribution:** generates particle packings, evaluates contact-chain counts, and exports simulation geometry.
- **Machine learning:** trains and validates regression models, predicts candidate formulation properties, and queries candidate formulations by target properties.

## 1. Repository contents

| File | Description |
|---|---|
| `particle_random_distribution.py` | Integrated application for particle quantity calculation, random packing, repeated contact-chain analysis, and STEP geometry export. |
| `geometry_cutpoints.csv` | Fixed particle geometry data, including particle categories, sizes, and vertex coordinates. |
| `geometry_volumes.csv` | Corresponding particle volume data used to calculate particle quantities. |
| `machine_learning.py` | Integrated application for model training, validation, candidate prediction, and formulation querying. |
| `Ml-data.csv` | Formulation-level dataset containing particle composition, simulated channel metrics, and simulation-derived elastic modulus values. |

The original component scripts are embedded in the two Python applications. Separate copies of those component scripts are not required.

## 2. Software requirements

The applications were developed for a Windows desktop environment. Python 3.11 with Tkinter is recommended.

### Core dependencies

Install the packages used for particle packing and machine learning:

```bash
python -m pip install numpy pandas scipy matplotlib tabulate scikit-learn joblib seaborn Pillow
```

Check that Tkinter is available:

```bash
python -m tkinter
```

This command should open a small test window.

### Additional dependencies for STEP export

The STEP export function additionally requires:

- `pyvista`
- `cadquery`
- `pythonocc-core`, providing the `OCC` Python package

These packages must be available in the Python environment used to launch the application. They are not required for the particle-count, contact-chain, or machine-learning workflows.

A fully pinned software environment is not currently included. The machine-learning application records installed package versions for each training run.

## 3. Particle random distribution

Start the application from the repository directory:

```bash
python particle_random_distribution.py
```

### Step 1: Load particle geometry

Click **Select geometry CSV files** and select both:

- `geometry_cutpoints.csv`
- `geometry_volumes.csv`

The selection field is initially blank. Selected filenames are displayed without their directory paths.

### Step 2: Define a custom formulation

Enter the masses, in grams, of particle categories A, B, C, and D.

Click **Calculate particle counts** to generate the particle quantity table for the selected formulation.

### Step 3: Generate batch formulations

As an alternative to the custom formulation:

1. Enter the total PLA mass.
2. Click **Generate 35 formulations**.
3. Select a group number.
4. Click **Use selected group**.

Steps 4–6 use the formulation confirmed in Step 2 or Step 3.

### Step 4: Run a single packing

Click **Run selected formulation**.

The application generates a random particle packing, performs contact-chain analysis, and saves the associated images and chain records.

### Step 5: Run repeated packings

Enter the number of repetitions and click **Run repetitions**.

Each repetition generates a new packing. Results are summarized in:

```text
iteration-all/main.csv
```

The summary contains:

| Column | Meaning |
|---|---|
| `iteration` | Repetition identifier. |
| `row_count` | Number of retained contact-chain records for that repetition. |
| `chain` | Associated contact-chain records. |

The `row_count` values can be averaged across repetitions to obtain a formulation-level mean channel metric. The current application does not automatically merge this average with modulus data to create the machine-learning training CSV.

### Step 6: Export simulation geometry

Click **Generate STEP files**.

This step generates a new random packing using the selected formulation. It does not reuse the particle positions from Steps 4 or 5.

Expected outputs include:

```text
polyhedron_data/
├── 1.polyhedrons.png
├── 2.polyhedrons.step
├── 3.space.step
└── 4.remaining_space.step
```

These files describe the particles, simulation domain, and remaining space after particle subtraction. This function exports geometry; finite-element solving and modulus calculation are performed separately.

### Output location

By default, particle workflow results are saved in timestamped subfolders under `GUI_runs`, next to the application. A different output folder can be selected in the interface.

The application retains step inputs, execution logs, and generated outputs. Use **Open latest results** to access the most recent output folder.

## 4. Machine-learning dataset

The supplied `Ml-data.csv` contains 210 formulation records: 35 compositions at each of six PLA mass levels, 12, 14, 16, 18, 20, and 22 g.

| Column | Meaning | Role |
|---|---|---|
| `exp_id` | Total PLA mass in grams. Despite its name, this field is used as total mass by the model. | Input |
| `w_a` | Fraction of particle category A. | Input |
| `w_b` | Fraction of particle category B. | Input |
| `w_c` | Fraction of particle category C. | Input |
| `w_d` | Fraction of particle category D. | Input |
| `channel_mean` | Mean simulated contact-chain/channel metric for the formulation. | Prediction target |
| `标准差` | Standard deviation associated with the channel metric. | Retained in the source dataset; not modeled as a target |
| `elastic_modulus (MPa)` | Simulation-derived elastic modulus in MPa. | Prediction target |

The four composition fractions must sum to 1.

The program derives 13 model features from the total mass and composition fractions. Derived features are calculated automatically.

The CSV reader supports multiple encodings, including UTF-8 and GBK. Keep the original column names when preparing a replacement dataset.

## 5. Model training and validation

Start the application:

```bash
python machine_learning.py
```

### Run the training workflow

1. Click **Browse** beside **Training CSV** and select `Ml-data.csv`.
2. Optionally select an output root folder.
3. Click **Train, validate and generate predictions**.
4. Wait for the completion message.

The training CSV field is initially blank. Selecting a file displays its filename without its directory path.

### Modeling procedure

The implemented workflow includes:

- Aggregation of repeated records sharing the same total mass and composition.
- An 80:20 training/test split with random seed 42.
- Ridge regression and gradient boosting as base learners.
- Eight-fold out-of-fold predictions within the training subset to construct stacking features.
- A random forest meta-regressor.
- Three-fold grid search for gradient-boosting hyperparameters.
- Strict nested tuning within the stacking folds.
- Additional eight-fold cross-validation within the training subset.
- Evaluation on the held-out test subset using R², RMSE, MSE, and MAE.

The two regression targets are modeled separately. Preprocessing is incorporated into the base-model pipelines.

The held-out test records come from the supplied computational dataset. This evaluation is an internal validation and does not constitute validation against new independent experiments.

The default nested validation workflow can be computationally intensive. Runtime depends on the dataset and available hardware.

## 6. Training outputs and candidate prediction

Each training run creates a separate timestamped directory under the selected output root. The default output root is `ML_runs`, next to the application.

Outputs include:

- A copy of the input dataset.
- `run_manifest.json`, containing execution status, software versions, model settings, and file checksums.
- Training/test split records and model evaluation results.
- Two trained stacking-model files in `.joblib` format.
- Figures and prediction records.
- `material_property_predictions.csv`.
- `training.log` when training is launched through the GUI.

Candidate prediction uses:

- Total PLA masses from 8 to 26 g, inclusive.
- Four-component fractions at increments of 0.05.
- 1,771 compositions per mass level.
- 33,649 candidate formulations in total.

The candidate space includes mass levels outside the 12–22 g range of the supplied training dataset. Predictions outside that range are extrapolations.

## 7. Query candidate formulations

After successful training, the application automatically selects the new prediction directory.

Click **Open formulation query** to open the query interface.

An existing directory containing `material_property_predictions.csv` can also be selected, allowing queries without retraining.

### Query by elastic modulus

Enter the target elastic modulus in MPa.

The program:

1. Selects candidates within ±0.00001 MPa of the target.
2. Ranks matching candidates by predicted channel metric in descending order.
3. Displays and exports up to 10 candidates.

The exported file is:

```text
elastic_modulus_test.csv
```

### Query by channel metric

Enter the target channel metric.

The program:

1. Selects candidates within ±0.001 of the target.
2. Ranks matching candidates by predicted elastic modulus in descending order.
3. Displays and exports up to 10 candidates.

The exported file is:

```text
channel_mean_test.csv
```

The lower table converts the highest-ranked candidate's composition fractions into particle masses.

The query interface searches the generated candidate prediction table. Its recommendations are limited to that candidate space and the specified query tolerance.

## 8. Command-line access

The machine-learning workflow also supports command-line execution.

Train, validate, and generate candidate predictions:

```bash
python machine_learning.py train --data Ml-data.csv --output ML_runs
```

Open the query interface for an existing prediction directory:

```bash
python machine_learning.py query --results "ML_runs/<run-folder>"
```

Replace `<run-folder>` with the actual timestamped directory name.

## 9. Interpretation and reproducibility

- The channel metric is derived from the implemented particle-contact and chain-processing rules. It is a computational indicator, not a direct measurement of three-dimensional pore connectivity.
- The supplied modulus values originate from simulation. The exported STEP files require a separate mechanical simulation workflow to produce modulus values.
- The particle workflow is stochastic. Repeated runs can produce different packings and channel counts.
- The machine-learning workflow uses a fixed random seed, but numerical results may still depend on software versions and the supplied dataset.
- Full nested model training, end-to-end STEP export, and manual GUI testing have not yet been completed for the integrated release. Completed checks include syntax, preservation of embedded source logic, dataset preprocessing, and candidate generation.
