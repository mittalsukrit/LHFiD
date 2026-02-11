# LHFiD

LHFiD is a **many-objective evolutionary algorithm (MaOEA)** implementation built on top of [`pymoo`](https://pymoo.org/). The repository contains a reusable algorithm class and a runnable notebook example for optimizing benchmark many-objective problems such as DTLZ.

The method corresponds to the algorithm described in the published paper:

- **Saxena, D., & Mittal, S.** “A New Many-Objective Evolutionary Algorithm Based on Localized Hyperplane-Following in Dominance (LHFiD).” *IEEE Transactions on Evolutionary Computation*.
- Paper link: https://ieeexplore.ieee.org/abstract/document/9814856

---

## Repository structure

- `LHFiD.py`: Core algorithm implementation (`LHFID`) and helper routines for survival, nadir estimation, and stabilization-based termination.
- `run_LHFiD.ipynb`: Minimal end-to-end notebook demonstrating problem setup, algorithm execution, and result visualization.
- `LICENSE`: Apache 2.0 license.

---

## Requirements

- Python **3.8+**
- `numpy`
- `pymoo`
- Jupyter (only if you want to run the notebook)

Install the main dependency:

```bash
pip install pymoo
```

If needed, also install notebook tooling:

```bash
pip install notebook
```

---

## Quick start (Notebook)

1. Keep `LHFiD.py` and `run_LHFiD.ipynb` in the same directory.
2. Launch Jupyter:

   ```bash
   jupyter notebook run_LHFiD.ipynb
   ```

3. Run all cells.

The notebook does the following:

- Imports `LHFID` from `LHFiD.py`
- Defines `dtlz4` with 3 objectives
- Creates Das-Dennis reference directions
- Runs `pymoo.optimize.minimize(...)` with a high generation cap
- Relies on LHFiD’s **internal stabilization-driven termination** to stop early
- Visualizes final objective vectors with `Scatter()`

---

## Quick start (Python script)

You can also run LHFiD without the notebook:

```python
from pymoo.problems import get_problem
from pymoo.util.ref_dirs import get_reference_directions
from pymoo.optimize import minimize
from pymoo.operators.selection.rnd import RandomSelection
from pymoo.operators.crossover.sbx import SBX
from pymoo.operators.mutation.pm import PM
from LHFiD import LHFID

problem = get_problem("dtlz4", n_obj=3, n_var=22)
ref_dirs = get_reference_directions("das-dennis", 3, n_partitions=13)

algorithm = LHFID(
    pop_size=len(ref_dirs),
    ref_dirs=ref_dirs,
    crossover=SBX(prob=0.9, eta=20),
    selection=RandomSelection(),
    mutation=PM(prob=1/problem.n_var, eta=20),
    eliminate_duplicates=True,
)

res = minimize(
    problem,
    algorithm,
    ("n_gen", 10000),  # large upper bound; LHFiD can terminate earlier
    seed=42,
)

print(res.F.shape)
```

---

## How LHFiD works (implementation overview)

At a high level, the implementation follows this loop:

1. **Initialization (`_initialize`)**
   - Generates initial population
   - Evaluates objective vectors
   - Computes the **ideal point** from the initial population
   - Leaves nadir point unset initially

2. **Offspring generation (`_infill`)**
   - Produces offspring via configured mating/crossover/mutation
   - Evaluates offspring
   - Updates the ideal point
   - Merges parent and offspring populations

3. **Environmental selection (`_advance` → `survival_selection`)**
   - Normalizes objective vectors if nadir exists, otherwise translates by ideal point
   - Associates solutions to reference vectors using perpendicular distance
   - Selects one solution per occupied vector using LHFiD’s comparison logic
   - Fills remaining vectors by closest unselected solutions

4. **Stabilization tracking + adaptive control**
   - Updates movement statistics (`mu_D`, `D_t`, `S_t`)
   - Triggers **nadir point update** when mild stabilization is detected
   - Triggers **termination suggestion** when stronger stabilization is detected

5. **Termination handling**
   - Once suggested, algorithm force-terminates
   - For 2D/3D objective cases, filters result to non-dominated set before final return

---

## Key parameters and defaults

Within `LHFID.__init__`, the following defaults are set unless overridden:

- `pop_size = len(ref_dirs)`
- Sampling: `FloatRandomSampling()`
- Crossover: `SBX(prob=0.9, eta=20)`
- Mutation: `PM(prob=0.1, eta=20)`
- Survival: `None` (custom selection inside LHFiD)
- Selection: `None` (user can provide, e.g., `RandomSelection()`)

Additional hard-coded control parameters in `_advance`:

- Nadir update check: `check_for_nadir_update(self, 2, 20)`
- Termination check: `check_for_termination(self, 3, 50)`

These correspond to decimal rounding precision and stabilization window length used by the tracking logic.

---

## Notes and practical tips

- The class name is `LHFID` (all caps for FID), imported from `LHFiD.py`.
- Provide suitable **reference directions** for your objective count.
- Keep the external generation cap high (e.g., `("n_gen", 10000)`) if you want LHFiD’s internal stabilization criterion to govern stopping.
- For reproducibility, pass a fixed `seed` to `minimize`.

---

## Troubleshooting

- **Import error for `pymoo`**: install with `pip install pymoo`.
- **Notebook cannot find `LHFiD`**: ensure notebook and `LHFiD.py` are in the same folder or adjust `PYTHONPATH`.
- **Unexpectedly long runs**: verify your reference vector count and problem difficulty; stabilization thresholds may take time to trigger.

---

## Citation

If you use this implementation in academic work, please cite the IEEE TEC LHFiD paper linked above.

---

## Contact

For help running this code:

- dhish.saxena@me.iitr.ac.in
- mittalsukrit@gmail.com
