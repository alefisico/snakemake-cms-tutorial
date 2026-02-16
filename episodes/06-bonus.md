---
title: 'Bonus: The CMSDAS Challenge'
teaching: 20
exercises: 40
---

:::::::::::::::::::::::::::::::::::::: questions

- How do I integrate existing CMS analysis repositories into Snakemake?
- How do I handle scripts that produce non-deterministic outputs (timestamps)?
- How do I chain different software environments (Coffea $\rightarrow$ Combine)?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Clone and configure the $t\bar{t}\gamma$ analysis repository.
- Write a rule to parallelize the processing step.
- "Patch" the aggregation step to accept Snakemake inputs.
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
git clone -b facilitators2026 https://github.com/fnallpc/ttgamma_longexercise.git
```

Now, check the contents. Notice that we are using the facilitators2026 branch (the solutions branch). You should see `runFullDataset.py` and a `ttgamma/` directory.

::::: callout

Notice that we are trying to give a "realistic" experience in this tutorial. The code is not designed for Snakemake, so we will have to make some adjustments and "patches" to make it work. This is a common scenario when integrating legacy code into modern workflows.
:::::::::::::::::::::::

### Step 1: The processing Rule

::::: spoiler

#### Patch 1: Removing Timestamps from `runFullDataset.py`

Open `ttgamma_longexercise/runFullDataset.py` and scroll to the bottom. You will see lines that look like this:

```python
    timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    outfile = os.path.join(args.outdir, f"output_{args.mcGroup}_run{timestamp}.coffea")
    util.save(output, outfile)
```

**The Problem:** Snakemake relies on filenames to know if a job finished. If the script adds a random timestamp (e.g., `_run20260215...`), Snakemake won't know the file was created, and the workflow will fail.

*The Fix*: Modify the code to remove the timestamp. Change those lines to:

```python
    # timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    # Remove the timestamp from the filename
    outfile = os.path.join(args.outdir, f"output_{args.mcGroup}.coffea")
    util.save(output, outfile)
```

Now the output is predictable: `output_MCTTGamma.coffea`.
:::::::::::::::::::::::::::::::::::::::::

We can write a clean Snakemake file with the following rules. We will define a rule that runs `runFullDataset.py` for each group (e.g., `MCTTGamma`, `Data`, etc.) and produces a `.coffea` file for each.

```python
# Define the groups (found in runFullDataset.py)
MC_GROUPS = ["MCTTGamma", "MCTTbar1l", "MCTTbar2l", "MCSingleTop", "MCZJets", "MCWJets", "MCOther"]
DATA_GROUPS = ["Data"]
ALL_GROUPS = MC_GROUPS + DATA_GROUPS

rule all:
    input:
        expand("results/output_{group}.coffea", group=ALL_GROUPS),
        "RootFiles/M3_Output.root"

rule run_coffea:
    input:
        script = "ttgamma_longexercise/runFullDataset.py"
    output:
        "results/output_{group}.coffea"
    container:
        #docker://coffeateam/coffea-dask-almalinux9:2025.9.0-py3.10"
        "/cvmfs/unpacked.cern.ch/registry.hub.docker.com/coffeateam/coffea-dask-almalinux9:2025.9.0-py3.10"   #### it can be from cvmfs
    shell:
        # We run with the modified script which outputs deterministic filenames
        "python {input.script} {wildcards.group} --outdir results --workers 1 --executor local --maxchunks 1"
        #"python {input.script} {wildcards.group} --outdir results --executor lpcjq"   ### if you want to run the full dataset, takes hours.
```

### Step 2: The Aggregation Patch

The second step of the analysis is to aggregate the `.coffea` files and convert them to `ROOT`.

::::: spoiler

#### Patch 2: Enabling Arguments in `save_to_root.py`

The original script `ttgamma_longexercise/save_to_root.py` has a major issue for automation and we will modify it.

```python
outputMC = accumulate(
    [
        util.load("results/output_MCOther.coffea"),
        util.load("results/output_MCSingleTop.coffea"),
        util.load("results/output_MCTTbar1l.coffea"),
        util.load("results/output_MCTTbar2l.coffea"),
        util.load("results/output_MCTTGamma.coffea"),
        util.load("results/output_MCWJets.coffea"),
        util.load("results/output_MCZJets.coffea"),
    ]
)

outputData = util.load("results/output_Data.coffea")
```

:::::::::::::::::::::::::::::::::

The second step of the analysis is to aggregate the `.coffea` files and convert them to ROOT. After you modify the code to remove the timestamp, we can write a Snakemake rule that depends on all the `.coffea` files and runs the aggregation script.

```python
rule make_root_files:
    input:
        # 1. We need ALL the group files to be finished
        coffeas = expand("results/output_{group}.coffea", group=ALL_GROUPS),
        # 2. The script we just created
        script = "ttgamma_longexercise/save_to_root.py"
    output:
        # The file expected by the next step (Combine)
        "RootFiles/M3_Output.root"
    container:
        # We use the same container as the processing step
        #docker://coffeateam/coffea-dask-almalinux9:2025.9.0-py3.10"
        "/cvmfs/unpacked.cern.ch/registry.hub.docker.com/coffeateam/coffea-dask-almalinux9:2025.9.0-py3.10"   #### it can be from cvmfs
    shell:
        # Pass the output filename first, then all input files
        "python {input.script} {output} {input.coffeas}"
```

### Step 3: The Fit (Hybrid Environments)

Now that we have ROOT files, we need to run `combine`. This requires a completely different environment (CMSSW-based).

While we usually use the `container`: directive, complex HEP software like CMSSW often requires sourcing environment scripts (`/cvmfs/.../cmsset_default.sh`) that don't play nicely with the automatic entrypoints of some containers.

In these cases, it is safer to invoke apptainer explicitly inside the shell command.

```python
rule run_combine:
    input:
        root_file = "RootFiles/M3_Output.root",
        # We use the data card from the repo
        card = "ttgamma_longexercise/Fitting/data_card.txt",
        #container = "docker://gitlab-registry.cern.ch/cms-cloud/combine-standalone:latest"
        container = '/cvmfs/unpacked.cern.ch/gitlab-registry.cern.ch/cms-analysis/general/combine-container:latest'
    output:
        "fitDiagnosticsTest.root"
    shell:
        """
        APPTAINER_SHELL=$(which bash) apptainer exec -B .:/home/cmsusr/analysis \
            -B /cvmfs --pwd /home/cmsusr/analysis/  \
            {input.container} \
                /bin/bash -c "export LANG=C && export LC_ALL=C && \
                source /cvmfs/cms.cern.ch/cmsset_default.sh && \
                cd /home/cmsusr/CMSSW_14_1_0_pre4/ && \
                 cmsenv && \
                 cd - && \
                text2workspace.py {input.card} -m 125 -o workspace.root && \
                combine -M FitDiagnostics workspace.root --saveShapes --saveWithUncertainties"
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

## Comparing Snakefile with a Bash Script

Let's compare this with how you would do it in a bash script.

::::: spoiler

### Full Snakefile for Reference

```python
# Define the groups (found in runFullDataset.py)
MC_GROUPS = ["MCTTGamma", "MCTTbar1l", "MCTTbar2l", "MCSingleTop", "MCZJets", "MCWJets", "MCOther"]
DATA_GROUPS = ["Data"]
ALL_GROUPS = MC_GROUPS + DATA_GROUPS

rule all:
    input:
        #expand("results/output_{group}.coffea", group=ALL_GROUPS),
        #"RootFiles/M3_Output.root",
        "fitDiagnosticsTest.root"

rule run_coffea:
    input:
        script = "ttgamma_longexercise/runFullDataset.py"
    output:
        "results/output_{group}.coffea"
    container:
        #docker://coffeateam/coffea-dask-almalinux9:2025.9.0-py3.10"
        "/cvmfs/unpacked.cern.ch/registry.hub.docker.com/coffeateam/coffea-dask-almalinux9:2025.9.0-py3.10"   #### it can be from cvmfs
    shell:
        # We run with the modified script which outputs deterministic filenames
        "python {input.script} {wildcards.group} --outdir results --workers 1 --executor local --maxchunks 1"
        #"python {input.script} {wildcards.group} --outdir results --executor lpcjq"

rule make_root_files:
    input:
        # 1. We need ALL the group files to be finished
        coffeas = expand("results/output_{group}.coffea", group=ALL_GROUPS),
        # 2. The script we just created
        script = "ttgamma_longexercise/save_to_root.py"
    output:
        # The file expected by the next step (Combine)
        "RootFiles/M3_Output.root"
    container:
        # We use the same container as the processing step
        #docker://coffeateam/coffea-dask-almalinux9:2025.9.0-py3.10"
        "/cvmfs/unpacked.cern.ch/registry.hub.docker.com/coffeateam/coffea-dask-almalinux9:2025.9.0-py3.10"   #### it can be from cvmfs
    shell:
        # Pass the output filename first, then all input files
        "python {input.script} {output} {input.coffeas}"


rule run_combine:
    input:
        root_file = "RootFiles/M3_Output.root",
        # We use the data card from the repo
        card = "ttgamma_longexercise/Fitting/data_card.txt",
        #container = "docker://gitlab-registry.cern.ch/cms-cloud/combine-standalone:latest"
        container = '/cvmfs/unpacked.cern.ch/gitlab-registry.cern.ch/cms-analysis/general/combine-container:latest'
    output:
        "fitDiagnosticsTest.root"
    shell:
        """
        #export APPTAINER_CACHEDIR="/tmp/$(whoami)/apptainer_cache"
        #export APPTAINER_TMPDIR="/tmp/.apptainer/"

        APPTAINER_SHELL=$(which bash) apptainer exec -B .:/home/cmsusr/analysis \
              -B /cvmfs --pwd /home/cmsusr/analysis/  \
              {input.container} \
              /bin/bash -c "export LANG=C && export LC_ALL=C && \
              source /cvmfs/cms.cern.ch/cmsset_default.sh && \
             cd /home/cmsusr/CMSSW_14_1_0_pre4/ && \
             cmsenv && \
             cd - && \
            text2workspace.py {input.card} -m 125 -o workspace.root && \
            combine -M FitDiagnostics workspace.root --saveShapes --saveWithUncertainties"
        """
```

:::::::::::::::::::::::::


:::::: spoiler

### Full Bash Script for Reference

```bash
#!/bin/bash

# ==============================================================================
# CONFIGURATION
# ==============================================================================
# Define the groups (mimicking the lists in your python script)
MC_GROUPS=("MCTTGamma" "MCTTbar1l" "MCTTbar2l" "MCSingleTop" "MCZJets" "MCWJets" "MCOther")
DATA_GROUPS=("Data")
ALL_GROUPS=("${MC_GROUPS[@]}" "${DATA_GROUPS[@]}")

# Define Container Paths
COFFEA_CONTAINER="/cvmfs/unpacked.cern.ch/registry.hub.docker.com/coffeateam/coffea-dask-almalinux9:2025.9.0-py3.10"
COMBINE_CONTAINER="/cvmfs/unpacked.cern.ch/gitlab-registry.cern.ch/cms-analysis/general/combine-container:latest"

# Fail on first error
set -e 

# ==============================================================================
# STEP 1: PROCESSING (Coffea)
# ==============================================================================
echo "--- Step 1: Running Coffea Processor ---"
mkdir -p results

# We have to manually loop over every sample
for group in "${ALL_GROUPS[@]}"; do
    output_file="results/output_${group}.coffea"
    
    # MANUAL CHECK: basic "target" logic (skip if exists)
    if [ -f "$output_file" ]; then
        echo "Skipping $group (Output exists)"
        continue
    fi

    echo "Processing $group..."
    
    # We have to manually invoke the container for EACH job
    apptainer exec -B .:$PWD $COFFEA_CONTAINER \
        python ttgamma_longexercise/runFullDataset.py \
        "$group" --outdir results --workers 1 --executor local --maxchunks 1
        
    # We still need the Rename/Move logic unless we patched the script
    # (Assuming we are using the patched script for this comparison)
done

# ==============================================================================
# STEP 2: AGGREGATION (Coffea)
# ==============================================================================
echo "--- Step 2: Aggregating Histograms ---"
mkdir -p RootFiles

# We have to construct the list of input files manually
INPUT_LIST=""
for group in "${ALL_GROUPS[@]}"; do
    INPUT_LIST="$INPUT_LIST results/output_${group}.coffea"
done

# Run the aggregation
# Note: We are using the patched/wrapper script we created
apptainer exec -B .:$PWD $COFFEA_CONTAINER \
    python scripts/make_root_wrapper.py \
    "RootFiles/M3_Output.root" $INPUT_LIST

# ==============================================================================
# STEP 3: STATISTICS (Combine)
# ==============================================================================
echo "--- Step 3: Running Fit ---"

# This requires the complex bind mounting and environment sourcing
APPTAINER_SHELL=$(which bash) apptainer exec -B .:/home/cmsusr/analysis \
    -B /cvmfs --pwd /home/cmsusr/analysis/ \
    $COMBINE_CONTAINER \
    /bin/bash -c "export LANG=C && export LC_ALL=C && \
    source /cvmfs/cms.cern.ch/cmsset_default.sh && \
    cd /home/cmsusr/CMSSW_14_1_0_pre4/ && \
    cmsenv && \
    cd - && \
    text2workspace.py ttgamma_longexercise/Fitting/data_card.txt -m 125 -o workspace.root && \
    combine -M FitDiagnostics workspace.root --saveShapes --saveWithUncertainties"

echo "Analysis Complete!"
```
:::::::::::::::::::::::::::::::::::

#### Key Differences to Highlight

* **Parallelism**:
    * Bash: Runs sequentially (one loop after another). To make this parallel, you'd need complex `&`, `wait`, or `GNU` parallel commands.
    * Snakemake: Just add `--cores 4` and it figures it out.

* **Resuming**:
    * Bash: Requires writing manual `if [ -f file ]; then ...` blocks. If you change a script, Bash won't know to re-run the step unless you delete the file manually.
    * Snakemake: Automatically detects if `runFullDataset.py` is newer than the output and re-runs only the necessary parts.

* **Environment Switching**:
    * Bash: You have to manually wrap every command in `apptainer exec ....`
    * Snakemake: You define the container: once per rule, and Snakemake handles the wrapping.

::::::::::::::::::::::::::::::::::::: keypoints 

- **Integration:** You can wrap almost any existing script in Snakemake, provided the Input/Output filenames are predictable.
- **Determinism:** If a script produces random timestamps or unique IDs in filenames, you must "patch" it to ensure Snakemake can track the files.
- **Hybrid Environments:** While `container:` is preferred, you can explicitly call `apptainer exec` inside a `shell` block when you need complex environment sourcing (like `cmsenv`).
- **Orchestration:** Snakemake can seamlessly connect completely different software stacks (e.g., Python/Coffea and C++/ROOT/Combine) into a single reproducible pipeline.

::::::::::::::::::::::::::::::::::::::::::::::::
