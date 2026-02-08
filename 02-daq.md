---
title: 'Chaining Rules (The DAG)'
teaching: 15
exercises: 15
---

:::::::::::::::::::::::::::::::::::::: questions

- How does Snakemake connect different rules?
- What is a DAG?
- How does Snakemake know what to re-run?
::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Connect two rules by matching input/output filenames.
- Use the `rule all` convention.
- Observe "Lazy Execution" in action.
::::::::::::::::::::::::::::::::::::::::::::::::

## Thinking Backwards

The most difficult paradigm shift when learning Snakemake is that you stop writing **Imperative** instructions (Step A, then Step B) and start writing **Declarative** goals.

In a Bash script, you say:
> "Run the skimmer. Then run the plotter."

In Snakemake, you say:
> "I want the plot. To get the plot, I need the skimmed file. To get the skimmed file, I need the raw data."

Snakemake determines the dependencies automatically by matching filenames. If Rule A outputs `file.txt` and Rule B takes `file.txt` as input, Snakemake knows Rule A must run first. This chain of dependencies is called a **DAG** (Directed Acyclic Graph).

## Activity: Extending the Analysis

We have a rule that skims data. Now we want to count the events in that skimmed file.

Add this **second rule** to your `Snakefile` (below the first one):

```python
rule count_events:
    input:
        "skimmed_data.txt" 
    output:
        "counts.txt"
    shell:
        "wc -l {input} > {output}"
```

**Crucial Link**: Notice that the `input` of `count_events` matches the `output` of `skim_data`. This is how Snakemake builds the **Directed Acyclic Graph (DAG)**.

## Running the Chain

Ask for the final result:

```bash
pixi run snakemake --cores 1 counts.txt
```

Snakemake realizes:

1. You want `counts.txt`.
2. `count_events` can produce it, but it needs `skimmed_data.txt`.
3. `skim_data` can produce `skimmed_data.txt`.
Plan: Run `skim_data` -> Run `count_events`.

## The rule all Convention

By default, Snakemake runs the **first rule** it sees if you don't specify a file. To avoid typing `counts.txt` every time, we add a "dummy" rule at the very top.

Add this to the **top** of your Snakefile:

```python
rule all:
    input:
        "counts.txt"
```

Now you can simply run:

```bash
pixi run snakemake --cores 1
```

:::: challenge

### Lazy Execution (The "Why")

1. Run `pixi run snakemake --cores 1` again.

What happens? 

```output
Assuming unrestricted shared filesystem usage.
host: xxxx
Building DAG of jobs...
Nothing to be done (all requested files are present and up to date).
```

2. Modify the original raw data:

```bash
touch raw_data.txt
```

3. Run Snakemake again.

What happens?

::::: solution

When you first run the command, Snakemake checks if `counts.txt` exists. Since it doesn't, it calculates the steps needed to create it. The second time you run the command, Snakemake sees that `counts.txt` exists and is newer than its inputs, so it does nothing.

When you "touch" `raw_data.txt`, you update its modification time. Snakemake notices that an input (`raw_data.txt`) is now strictly newer than the downstream files (`skimmed_data.txt` and `counts.txt`). It marks them as "stale" and re-runs the chain.

**This is the crucial benefit for large analyses.**
If you had a workflow with 500 rules and you only modified the input for rule 499, Snakemake would **not** re-run rules 1 through 498. It selectively re-executes only the parts of the DAG that are affected by your change. In CMS terms: if you change a plotting style, you don't have to re-run the N-tuplizer.

::::::::::
::::::::::

::::::::::::::::::::::::::::::::::::: keypoints

* **Declarative Workflows**: Unlike bash scripts where you define the *order* of steps, in Snakemake you define the *dependencies* (inputs/outputs), and Snakemake figures out the order (DAG).
* **The `all` Rule**: It is convention to include a rule named `all` at the top of the workflow to define the final targets of your analysis.
* **Lazy Execution**: Snakemake only re-runs a rule if the output file is missing or if the input files have changed (have a newer timestamp) since the last run.
::::::::::::::::::::::::::::::::::::::::::::::::
