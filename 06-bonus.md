---
title: 'Bonus: The CMSDAS Challenge'
teaching: 20
exercises: 40
---

:::::::::::::::::::::::::::::::::::::: questions 

- How do I integrate existing CMS analysis repositories into Snakemake?
- How do I handle scripts that produce non-deterministic outputs (timestamps)?
- How do I chain different software environments (Coffea -> Combine)?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Clone and configure the $t\bar{t}\gamma$ analysis repository.
- Write a rule to parallelize the `runFullDataset.py` processor.
- "Patch" the aggregation step to accept Snakemake inputs instead of hardcoded strings.
- Execute a final statistical fit using a dedicated `combine` container.

::::::::::::::::::::::::::::::::::::::::::::::::

## The Challenge: $t\bar{t}\gamma$ Cross Section

In the previous episodes, we worked with toy scripts. Now, we will automate a real analysis: the **CMSDAS $t\bar{t}\gamma$ Long Exercise**. For this tutorial, we are not interested about the physics details, but rather how to integrate an existing analysis workflow into Snakemake. If you want to know more about the physics, check the [CMSDAS $t\bar{t}\gamma$ Long Exercise repository](https://github.com/FNALLPC/TTGamma_LongExercise).

This analysis has a typical structure:

1. **Coffea Processor (`runFullDataset.py`):** Runs on NanoAODs and produces `.coffea` histograms.
2. **Plotting/Conversion (`save_to_root.py`):** Aggregates histograms and saves them as `.root` files for the fit.
3. **Statistics (`combine`):** Performs a likelihood fit to extract the cross-section.

### Setting the Stage

First, we need the analysis code. We will clone the repository directly into our workflow directory.

```bash
# Clone the repository (using the facilitators2026 branch for this tutorial)
git clone -b facilitators2026 [https://github.com/fnallpc/ttgamma_longexercise.git](https://github.com/fnallpc/ttgamma_longexercise.git)
```

Now, check the contents. Notice that we are using the facilitators2026 branch (the solutions branch). You should see `runFullDataset.py` and a `ttgamma/` directory.

### Step 1: Taming the Processor

Open `ttgamma_longexercise/runFullDataset.py`. It takes an argument `mcGroup` (like `MCTTGamma`, `Data`, `MCTTbar1l`) and outputs a file with a timestamp, e.g., `output_MCTTGamma_run20260215_120000.coffea`.

**The Problem**: Snakemake needs to know the exact output filename to build the DAG. If the script adds a random timestamp, Snakemake will crash saying "Output file not found."

**The Solution**: We will write a rule that runs the script and then *moves/renames* the output to a deterministic name.

Create a new Snakefile with this content:

```python
# Define the groups (found in runFullDataset.py)
MC_GROUPS = ["MCTTGamma", "MCTTbar1l", "MCTTbar2l", "MCSingleTop", "MCZJets", "MCWJets", "MCOther"]
DATA_GROUPS = ["Data"]
ALL_GROUPS = MC_GROUPS + DATA_GROUPS

rule run_coffea:
    output:
        "results/{group}.coffea"
    container:
        "docker://coffeateam/coffea-dask:latest"
    shell:
        """
        # 1. Run the script (it writes to 'Outputs/' by default)
        python ttgamma_longexercise/runFullDataset.py {wildcards.group} --outdir results --workers 1 --executor local
        
        # 2. The script adds a timestamp. We need to rename it to match our output.
        # Find the file that looks like output_{group}_run*.coffea and move it.
        mv results/output_{wildcards.group}_run*.coffea {output}
        """
```

:::: discussion

#### Why did we move the file? 

Because Snakemake is a file-based system. If the file we promised (`results/MCTTGamma.coffea`) isn't created, the pipeline fails.
::::::::::::::::

### Step 2: The Aggregation Patch

The original script `ttgamma_longexercise/save_to_root.py` has a major issue for automation: **Hardcoded Filenames**.

```python
# OLD CODE in save_to_root.py
outputMC = accumulate([
    util.load("Outputs/output_MCOther_run20240103_184531.coffea"),
    ...
])
```

We cannot rely on hardcoded dates! We need a script that accepts our clean filenames from Step 1.

Instead of modifying the original code (which is bad practice if we want to pull updates later), let's create a new script `scripts/make_root_snakemake.py` that imports the logic but lets us pass filenames.

Create `scripts/make_root_snakemake.py`:

```python
import sys
from coffea import util
# Import specific functions from the analysis repo
# We need to add the repo to python path
sys.path.append("ttgamma_longexercise")
from save_to_root import groupingMCDatasets, groupingCategory  # Reuse their config!

# 1. Load inputs from command line arguments (provided by Snakemake)
# Usage: python make_root.py output.root input1.coffea input2.coffea ...
output_filename = sys.argv[1]
input_files = sys.argv[2:]

print(f"Aggregating {len(input_files)} files into {output_filename}...")

# 2. Accumulate
merged_output = util.load(input_files[0])
for fname in input_files[1:]:
    merged_output += util.load(fname)

# ... (Insert the rest of the Histogram -> ROOT logic here) ...
# For this tutorial, we will cheat slightly and just save the raw merged object 
# to demonstrate the workflow connection, or you can copy the logic from save_to_root.py
```

:::: instructor

For the actual tutorial, provide the full content of this wrapper script as a downloadable file or a collapsible code block so they don't have to debug Python imports.
:::::::::::::

Now write the rule that calls this wrapper.

```python
rule make_root_files:
    input:
        # Request ALL coffea files to be done
        coffeas = expand("results/{g}.coffea", g=ALL_GROUPS),
        script = "scripts/make_root_snakemake.py"
    output:
        "RootFiles/final_shapes.root"
    container:
        "docker://coffeateam/coffea-dask:latest"
    shell:
        "python {input.script} {output} {input.coffeas}"
```

### Step 3: The Fit (Switching Containers)

Now that we have ROOT files, we need to run `combine`. This requires a completely different environment (CMSSW-based).

The `combine` tool is available in a standalone container, so we can just switch containers for this step. However, there are a few settings we need to be careful about:

```python
rule run_combine:
    input:
        root_file = "RootFiles/final_shapes.root",
        # We use the data card from the repo
        card = "ttgamma_longexercise/Fitting/data_card.txt"
    output:
        "fitDiagnosticsTest.root"
    container:
        "docker://gitlab-registry.cern.ch/cms-cloud/combine-standalone:latest"
    shell:
        """
        # Combine often requires text2workspace first
        text2workspace.py {input.card} -m 125 -o workspace.root
        
        # Run the fit
        combine -M FitDiagnostics workspace.root --saveShapes --saveWithUncertainties
        """
```

### The Grand Finale

Now, define your target in rule all.

```python
rule all:
    input:
        "fitDiagnosticsTest.root"
```

Run it!

```bash
pixi run snakemake --cores 4 --use-apptainer
```

:::: callout

### What just happened?

1. Snakemake saw you wanted the Fit.
2. It checked `make_root_files`, which needed the `.coffea` files.
3. It launched 8 parallel jobs to process `Data`, `TTGamma`, `TTbar`, etc., using the Coffea container.
4. Once all 8 finished, it ran the aggregation script.
5. Finally, it switched to the Combine container and performed the fit.

You have just orchestrated a full CMS analysis involving Data, MC, Systematics, and Statistics with one command.

:::::::::::::::::::::::::::::::::::::::::::::::
