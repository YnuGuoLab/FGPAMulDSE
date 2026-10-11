# FPGA Multiplier Design and Circuit-Aware Design Space Exploration

This repository contains the Verilog implementations, Python evaluation scripts, and experimental data for **Carry-Optimized Low-Area FPGA Multipliers and Circuit-Aware Design Space Exploration**.

The framework combines low-area exact and approximate 4 × 4 multipliers, matrix-based error evaluation, and circuit-aware hardware prediction to explore composed 8 × 8 architectures.

## Repository Structure

```text
FGPAMulDSE/
└── Project/
    ├── 4bit/
    │   ├── Proposed/                  # LCAM and BFAM1–BFAM3
    │   ├── Approx_Mul_FPGA/           # Accurate, LSAM, and CSAM references
    │   ├── LM/                        # LM reference implementations
    │   ├── CA/                        # CA comparison project
    │   └── data-4.xlsx                # Hardware and error results
    ├── 8bit/
    │   ├── Proposed.zip              # 140 Vivado projects and recorded results
    │   └── data-8.xlsx                # Measured results for 140 architectures
    └── DSE/
        ├── ErrorCalculation/
        │   ├── add-matrix-mapping.py
        │   ├── add-matrix-mapping/    # Cached adder lookup arrays
        │   ├── 8bit-error-calculation.py
        │   └── 8bit-error.xlsx
        └── CAML/
            ├── CARFforDELAY.py
            ├── CAGPRforPOWER.py
            └── Prediction Result.xlsx
```

The tree describes the supplied package. The project archive and cached arrays may be distributed separately as release assets because of their size. Cached arrays can also be regenerated with the supplied script.

## Environment

- **FPGA:** AMD/Xilinx Virtex-7, `xc7vx330tffg1157-1`.
- **HDL tools:** Vivado 2019.1, including XSim and the Xilinx primitive simulation libraries.
- **Software environment reported for the experiments:** Python 3.12.7, NumPy 1.26.4, Numba 0.61.2, scikit-learn 1.5.1, and SciPy 1.12.0.
- **Additional dependencies:** pandas and joblib; openpyxl is needed to convert the Excel data.

```bash
python -m pip install numpy==1.26.4 numba==0.61.2 scikit-learn==1.5.1 scipy==1.12.0 pandas joblib openpyxl
```

Run the commands below from the repository root. The current scripts contain absolute Windows paths; configure them for your machine before execution.

## 1. 4 × 4 Multipliers

### Proposed Designs

All four designs have unsigned 4-bit inputs `a` and `b` and an 8-bit output `P`.

| ID | Design | File in `4bit/Proposed/` | Verilog module | Module in the 8-bit archive | Lookup-table filename |
| --- | --- | --- | --- | --- | --- |
| 1 | LCAM, exact | `LCAM.v` | `Acc4X4` | `AC` | `Ac.txt` |
| 2 | BFAM1, approximate | `BFAM1.v` | `AMC1` | `A1` | `AMC1.txt` |
| 3 | BFAM2, approximate | `BFAM2.v` | `AMC2` | `A2` | `AMC2.txt` |
| 4 | BFAM3, approximate | `BFAM3.v` | `NCC` | `A3` | `NCC.txt` |

Instantiate the **Verilog module name**, rather than the filename. The archived modules use different names but implement the corresponding standalone designs.

### Reference Designs

- `Approx_Mul_FPGA/`: `accurate4.v`, `LSAM1.v`, `LSAM2.v`, `CSAM1.v`, and `CSAM2.v`.
- `LM/`: `LMACC.v`, `LM1.v`, `LM2.v`, `LM3.v`, and `LMNC.v`.
- `CA/`: a Vivado comparison project that currently references external VHDL files outside this package. Supply those files and repair their paths before rebuilding that project.

The module names in the first two reference directories match their filenames without `.v`. Check each module's port names before connecting a testbench.

### Simulation and Synthesis

1. Create a Vivado RTL project for `xc7vx330tffg1157-1`.
2. Add the selected design source and instantiate it in a simulation testbench.
3. Enumerate all **256** unsigned input pairs, with both inputs ranging from 0 to 15, for functional evaluation.
4. Run behavioral simulation with the Xilinx primitive libraries. Run synthesis and implementation separately for hardware evaluation.

The matrix evaluator requires the four lookup-table files listed above. Each row must contain whitespace-separated binary values for `a`, `b`, and `P`, with widths of 4, 4, and 8 bits. For example:

```text
0000 0000 00000000
0000 0001 00000000
...
```

Generate all 256 rows for each design and save the four files in `Project/DSE/ErrorCalculation/multiplier-tables/`. This directory and these text files need to be created; they are not included in the current package. Preserve the filename capitalization shown in the table.

## 2. Composed 8 × 8 Multipliers

Each architecture combines four 4 × 4 multipliers with one accumulation module. Four adder families are included: **ACCA**, **MODA**, **HSLP**, and **NCCA**.

### Naming Convention

The name `<ADDER><HH><HL><LH><LL>` specifies the adder and four multiplier IDs:

| Position | Input pair | Output weight |
| --- | --- | --- |
| HH | `a[7:4] × b[7:4]` | `2^8` |
| HL | `a[7:4] × b[3:0]` | `2^4` |
| LH | `a[3:0] × b[7:4]` | `2^4` |
| LL | `a[3:0] × b[3:0]` | `1` |

For example, `ACCA1234` uses ACCA with LCAM, BFAM1, BFAM2, and BFAM3 at HH, HL, LH, and LL, respectively. Its top-level module is `ACCA1234`.

The supplied set contains 35 nondecreasing digit combinations per adder, giving **140 architectures**. This is a selected design space; all ordered assignments would give 1,024 architectures.

### Using the Vivado Projects

1. Extract `Project/8bit/Proposed.zip` into `Project/8bit/Proposed/`.
2. Open a project such as `Proposed/ACCA1234/ACCA1234.xpr`.
3. Use the design sources and testbench bundled with that project.
4. Update the input and output paths in `tb_Product_Generation.v`. Its `input8.txt` must contain all 256 unsigned 8-bit values in ascending order, one binary value per line.
5. Run simulation over all **65,536** input pairs. Run synthesis and implementation for hardware evaluation.

Generate the input file with:

```python
from pathlib import Path

Path("input8.txt").write_text(
    "".join(f"{value:08b}\n" for value in range(256)), encoding="ascii",
)
```

Use one architecture project at a time. Adding all project sources together would introduce duplicate module definitions such as `AC`, `A1`, and `ACCA`.

## 3. Matrix-Based Error Evaluation

`add-matrix-mapping.py` generates reusable lookup arrays for the four accumulation modules. `8bit-error-calculation.py` combines these arrays with the 4-bit lookup tables to calculate ER, MED, NED, and MRED.

### Configure the Paths

The supplied `.npy` arrays are stored directly in `add-matrix-mapping/`, whereas the original evaluator expects an additional subdirectory for each adder. To use the supplied flat directory, add `from pathlib import Path` to the evaluator and replace the three path assignments at the start of `run_evaluation()` with:

```python
base_dir = Path(__file__).resolve().parent
mult_dir = str(base_dir / "multiplier-tables")
adder_root_dir = str(base_dir / "add-matrix-mapping")
output_dir = str(base_dir / "results")
```

Replace the construction of `matrix_path` in the adder-loading loop with:

```python
matrix_path = os.path.join(adder_root_dir, f"{add_n.lower()}_matrix.npy")
```

Confirm that all four matrix files and all four multiplier text files are present. The current evaluator skips an adder if its matrix is missing.

To regenerate the arrays, set all four output directories in the generator's `tasks` list to the same flat cache directory, then run:

```bash
python Project/DSE/ErrorCalculation/add-matrix-mapping.py
```

Each array contains `2^28` entries of type `uint16`, requiring approximately **512 MiB**. The four arrays occupy approximately **2 GiB** of disk space and, when loaded together by the current evaluator, roughly the same amount of RAM before other allocations.

### Run the Evaluation

```bash
python Project/DSE/ErrorCalculation/8bit-error-calculation.py
```

The active block evaluates the 140-architecture set and writes `results/02_logical_140_combinations.csv`. Custom-architecture and 1,024-architecture evaluation blocks are commented out by default.

| Metric | Definition and units |
| --- | --- |
| ER | Fraction of input pairs with nonzero absolute error; range 0–1 |
| MED | Mean absolute error over all 65,536 input pairs |
| NED (%) | `100 × MED / 65025`, where `65025 = 255 × 255` |
| MRED (%) | Mean relative absolute error × 100, excluding input pairs with an exact product of zero |

## 4. Circuit-Aware Hardware Prediction

The CAML scripts use measured characteristics of the four position-ordered submultipliers and the accumulation module:

| Script | Target | Inputs and model |
| --- | --- | --- |
| `CARFforDELAY.py` | Delay in ns | Ten module delay/power features; RF leaf encoding followed by ridge regression |
| `CAGPRforPOWER.py` | Logic power in mW | The ten module features and total LUT count; a Matérn 3/2 kernel, a linear resource term, and a noise term |

Each script selects representative training samples using K-means with 30 clusters. The experimental split contains **30 training architectures and 110 test architectures**. Selection is performed separately by the two scripts using their respective feature sets.

### Prepare the Measured Data

Both scripts expect a CSV with `Project_Name`, `Max_Delay_ns`, and `Logic_Power_mW`. Convert the supplied measured data as follows:

```python
from pathlib import Path
import pandas as pd

root = Path.cwd()  # Run from the repository root.
df = pd.read_excel(root / "Project/8bit/data-8.xlsx", header=1)
df = df.rename(columns={
    "Delay (ns)": "Max_Delay_ns",
    "Power (mW)": "Logic_Power_mW",
}).dropna(subset=["Project_Name"])
df[["Project_Name", "Max_Delay_ns", "Logic_Power_mW"]].to_csv(
    root / "Project/DSE/CAML/summary_results.csv", index=False,
)
```

Use measured Vivado results as training labels. `Prediction Result.xlsx` is a prediction-result table and should not replace the measured data.

In **both** model scripts, replace `NEW_DATA_FILE`, `OUTPUT_DATA_DIR`, and `OUTPUT_MODEL_DIR` with:

```python
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
NEW_DATA_FILE = os.path.join(BASE_DIR, "summary_results.csv")
OUTPUT_DATA_DIR = os.path.join(BASE_DIR, "results")
OUTPUT_MODEL_DIR = os.path.join(BASE_DIR, "models")
```

Then run:

```bash
python Project/DSE/CAML/CARFforDELAY.py
python Project/DSE/CAML/CAGPRforPOWER.py
```

The scripts save fitted models and prediction tables. Some output names retain the earlier PIML terminology:

- **CARF:** `PIML-RF_Max_Delay_ns.pkl` and `PIML-RF_Delay_Predicted_Results.csv`.
- **CAGPR:** `PIML_GPR_LogicPower.pkl`, `GPR_Predicted_Results.csv`, and `Area_Predicted_Results.csv`.
- `Uncertainty_Std` is the power prediction's standard deviation, in mW.
- The area table contains LUT and CARRY4 counts summed from component characteristics.

Join error and prediction results by architecture name to compare accuracy–hardware trade-offs. The supplied scripts do not automatically generate the paper's Pareto-front figures.

## Experimental Data

| File | Contents |
| --- | --- |
| `Project/4bit/data-4.xlsx` | Hardware and error results for proposed and reference 4-bit designs |
| `Project/8bit/data-8.xlsx` | Measured hardware and error results for 140 composed architectures |
| `Project/DSE/ErrorCalculation/8bit-error.xlsx` | Matrix-based error results for 140 architectures |
| `Project/DSE/CAML/Prediction Result.xlsx` | Supplied hardware-prediction result table |

Delay is in **ns**, logic power in **mW**, and resource usage in **LUT/CARRY4 counts**. Keep the ER fraction and NED/MRED percentage conventions consistent when comparing results.

## Acknowledgments and Citation

The reference-design directory `Project/4bit/Approx_Mul_FPGA/` corresponds to the accurate, LSAM, and CSAM family available in [YnuGuoLab/Approx_Mul_FPGA](https://github.com/YnuGuoLab/Approx_Mul_FPGA). These reference implementations are distinct from the proposed LCAM and BFAM designs.

When using this repository in research, please cite the accompanying paper and the original publications for any reused reference implementations. Publication details and a BibTeX entry for the accompanying paper will be added when available.

## License

The supplied package currently has no repository-level license file. License information for the original contributions remains to be specified; reused implementations retain their original license terms.
