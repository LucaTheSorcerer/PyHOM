# py_hom.py — 2D Plate Vibration Analysis

CLI tool that estimates how an added mass patch changes the natural frequencies of a **clamped–clamped rectangular plate**.

It uses a physics-based **Rayleigh / modal-mass** approach: mode shapes are kept fixed, and the frequency shift comes from the change in modal mass when a denser patch is placed on the plate.

Two modes of use:

1. **Interactive** — enter one plate / patch / mode case by hand
2. **Batch** — read many FEM validation cases from Excel and write formatted results

---

## What it does

Given plate size, patch location, vibration mode `(i, j)`, a healthy-plate reference frequency, and a mass ratio in the patch region, the script:

1. Builds 1D clamped–clamped beam mode shapes for axes $X$ and $Y$
2. Forms the 2D squared mode-shape surface $\Phi^2(x,y) = \phi_i^2(x)\,\phi_j^2(y)$
3. Numerically integrates that surface over the full plate and over the defect patch
4. Computes the frequency with added mass via the modal-mass ratio

This is a fast analytical proxy for FEM: you get frequency estimates in seconds instead of running a full structural simulation.

---

## Method (short)

Natural frequency scales with modal mass under a pure mass perturbation (stiffness and mode shape assumed unchanged):

$$
\frac{f_{\text{defect}}}{f_{\text{healthy}}} = \sqrt{\frac{M^*_{\text{healthy}}}{M^*_{\text{defect}}}}
$$

Modal mass is obtained from the double integral of the squared mode shape (trapezoidal rule on a $101 \times 101$ grid). In the patch region, density is scaled by the **mass ratio**.

Clamped–clamped beam eigenvalues $\lambda$ for modes 1–6 are built into the script.

For a fuller derivation, see [`Integral_Method_Explanation.md`](Integral_Method_Explanation.md).

---

## Requirements

- Python 3
- NumPy
- openpyxl (batch mode only)

```bash
pip install numpy openpyxl
```

---

## Usage

### Interactive mode

```bash
python py_hom.py
```

The program prompts for all inputs.


### Batch mode (Excel)

Process FEM validation cases from an input workbook and write a formatted results file:

```bash
python py_hom.py --batch Correct_data.xlsx --output Initial_results_Formatted.xlsx
```

| Flag       | Default                          | Meaning                   |
|------------|----------------------------------|---------------------------|
| `--batch`  | *(required for batch)*           | Path to input Excel file  |
| `--output` | `Initial_results_Formatted.xlsx` | Path to output Excel file |

#### What batch mode does

1. Opens the input workbook (`Sheet1`)
2. Reads **20 mode labels** and healthy frequencies from the header rows
3. Reads **4 test cases** (patch location + FEM frequencies)
4. For each test case × mode, computes the Python integral frequency
5. Writes absolute error vs FEM into a formatted `Validation Results` sheet

Fixed batch assumptions (matching the validation setup):

- Plate size: $X_1 = 3\,\mathrm{m}$, $X_2 = 2\,\mathrm{m}$
- Mass ratio: $15000 / 7850$ (added patch density vs steel)

#### Expected input Excel layout (`Sheet1`)

| Location                  | Content                                                 |
|---------------------------|---------------------------------------------------------|
| Row 2, columns D–W (4–23) | Mode labels as `"i-j"` (e.g. `1-1`, `1-2`, …)           |
| Row 3, columns D–W        | Healthy-plate frequencies (Hz) for those modes          |
| Rows 5–8, column A        | Test-case name                                          |
| Rows 5–8, column B        | Patch X range as `"A1-B1"` in metres (e.g. `0.32-0.41`) |
| Rows 5–8, column C        | Patch Y range as `"A2-B2"` in metres                    |
| Rows 5–8, columns D–W     | FEM defect frequencies (Hz) for each mode               |

#### Output Excel columns

| Column                                         | Description                    |
|------------------------------------------------|--------------------------------|
| Healthy / Defect Plate Dimensions              | Plate size labels              |
| Test Case                                      | Case name from the input sheet |
| Defect Start X/Y (mm)                          | Patch start converted to mm    |
| A1, B1, A2, B2 (m)                             | Patch bounds                   |
| Mode (i-j)                                     | Mode pair                      |
| Freq Healthy (Hz)                              | Intact reference frequency     |
| Freq Defect FEM (Hz)                           | FEM target from the input file |
| Python Integral Frequency with added mass (Hz) | Script prediction              |
| Absolute Error (Hz)                            | \|FEM − Python\|               |

The output is compatible with [`plot_validation.py`](plot_validation.py) for parity / heatmap plots.

---

## Interactive inputs

| Prompt             | Meaning                                                      |
|--------------------|--------------------------------------------------------------|
| `X1`, `X2`         | Plate length and width                                       |
| `A1`, `B1`         | Patch start/end along $X$                                    |
| `A2`, `B2`         | Patch start/end along $Y$                                    |
| `mode i`, `mode j` | Mode indices along $X$ and $Y$ (eigenvalues defined for 1–6) |
| `f_ref`            | Healthy-plate reference frequency (Hz)                       |
| `mass_ratio`       | Density multiplier in the patch (`> 1` = heavier added mass) |

Patch bounds are sorted automatically if entered out of order.

---

## Interactive output

The CLI prints the **physics integral** results, including:

- Total plate integral (intact modal participation)
- Defect-region integral
- Plate area and average participation per m²
- Frequency ratio and the estimated frequency **with** and **without** the added mass

---

## Core functions

| Function                       | Role                                                               |
|--------------------------------|--------------------------------------------------------------------|
| `calculate_squared_mode_shape` | Normalized 1D clamped–clamped $\phi^2$ along one axis              |
| `integrate_region`             | 2D trapezoidal integral of $\Phi^2$ over a rectangular region      |
| `compute_freq_integral`        | End-to-end frequency estimate for a given mode and patch           |
| `process_batch_excel`          | Read validation cases from Excel, compute all modes, write results |
| `main`                         | CLI entry point (interactive or `--batch`)                         |

---

