---
title: 'Scaling with Wildcards'
teaching: 15
exercises: 15
---

:::::::::::::::::::::::::::::::::::::: questions

- How can I use one rule to process multiple different samples?
- What is a wildcard and how does Snakemake "fill" it?
- How do I tell Snakemake to generate a list of all my target files?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Replace hardcoded filenames with `{wildcards}`.
- Use the `expand()` function to generate lists of outputs.
- Understand how Snakemake "pattern matches" files on disk.

::::::::::::::::::::::::::::::::::::::::::::::::

## Scaling Up: From One File to Many

In CMS, we never have just one "raw_data.txt". We have `DYJets`, `TTbar`, `WJets`, and various `Data` eras. Writing a rule for each one would be a nightmare.

Snakemake handles this using **Wildcards**.

---

## The Wildcard Syntax

A wildcard is a placeholder in curly braces `{}`. For example, instead of writing a rule that only processes `TTbar.txt`, we can write a generic rule that works for any sample:

```python
rule skim_data:
    input:
        "raw/{sample}.txt"
    output:
        "skimmed/{sample}.txt"
    shell:
        "grep 'Signal' {input} > {output}"
```

### How does it work?

When you ask for `skimmed/TTbar.txt`, Snakemake looks at the rule and sees it can create `skimmed/{sample}.txt`. It "pattern matches" and determines that `{sample}` must be `TTbar`. It then looks for the input `TTbar.txt`.

**Crucial Rule**: Snakemake works **backwards**. It looks at the output you requested, matches it to the output pattern of a rule, determines the wildcard value, and then fills in that value for the input.

---

## The `expand()` function

If you have 100 samples, you don't want to type them all in your `rule all`. Snakemake provides a helper function called `expand()` to generate lists of files.

```python
SAMPLES = ["DYJets", "TTbar", "WJets"]

rule all:
    input:
        expand("skimmed/{s}.txt", s=SAMPLES)
```

The `expand()` function takes a pattern and replaces the placeholders with the values in your list. The code above produces: `["skimmed/DYJets.txt", "skimmed/TTbar.txt", "skimmed/WJets.txt"]`

::: callout

### Wildcards vs. Expand Variables

Notice a subtle difference:

1. In `rule skim_data`, we used `{sample}`. This is a **Wildcard** (Snakemake figures it out based on the filename).
2. In `rule all`, we used `{s}` inside `expand()`. This is a Python string formatting variable.

They do not need to match! `expand()` happens before the rules run to generate the list of target files. The rules run after to figure out how to create those files.

:::::::::::::

---

## Activity: Processing Multiple Datasets

:::: instructor

* **Parallelism:** This is the best moment to explain why the `--cores` flag matters. In HEP, we are used to sending 100 jobs to Condor. Here, we show they can run 4 (or 8, or 16) jobs in parallel locally on their laptop with zero extra effort.
* **The "Pattern Matching" Warning:** Students often try to put wildcards in the `input` that aren't in the `output`. I would emphasize that Snakemake works **backwards**: it sees a file it wants (the output) and then tries to figure out what the input should be.

::::::::

Let's modify our `Snakefile` to handle three different simulated datasets.

1. Open your `Snakefile` and modify it as follows:

```python
# 1. Define our datasets
DATASETS = ["DYJets", "TTbar", "Data"]

rule all:
    input:
        expand("results/{d}_counts.txt", d=DATASETS)

# 2. Updated Skim rule with wildcards
rule skim_data:
    input:
        "raw/{dataset}.txt"
    output:
        "skimmed/{dataset}.txt"
    shell:
        "grep 'Signal' {input} > {output}"

# 3. Updated Count rule with wildcards
rule count_events:
    input:
        "skimmed/{dataset}.txt"
    output:
        "results/{dataset}_counts.txt"
    shell:
        "wc -l {input} > {output}"
```

2. Prepare the "raw" directory and files:

```bash
mkdir -p raw
echo -e "Signal\nBackground" > raw/DYJets.txt
echo -e "Signal\nSignal\nBackground" > raw/TTbar.txt
echo -e "Background\nBackground" > raw/Data.txt
```

3. Run the workflow. Note that we use --cores 4 to allow Snakemake to run independent jobs in parallel:

```bash
pixi run snakemake --cores 4
```

:::: challenge

### Exercise: Adding a new sample

Add a new dataset called `WJets` to your `DATASETS` list.

1. Create the dummy file `raw/WJets.txt` with some "Signal" lines.
2. Run Snakemake again.

:::: solution

Observe how Snakemake only runs the rules for the new `WJets` sample and skips the ones that were already finished (`DYJets`, `TTbar`, `Data`).

::::::::::
::::::::::

:::::::::::::::::::::::::::::::::::: keypoints

* **Wildcards**: Use `{name}` in filenames to define a generic rule.
* **Constraints**: Snakemake fills wildcards by looking at the *output* you requested and propagating that value to the *input*.
* **expand()**: A Python function that generates a list of filenames from a pattern. It is commonly used in `rule all` to define the final targets.
* **Parallelism**: With wildcards, Snakemake can run multiple independent jobs in parallel using the `--cores` flag.

::::::::::::::::::::::::::::::::::::::::::::::::
