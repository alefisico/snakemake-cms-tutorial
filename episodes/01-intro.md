---
title: 'Introduction to Snakemake'
teaching: 10
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions

- What is a Snakefile?
- How do I define a single processing step?
- How do I execute a rule?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Create a `Snakefile`.
- Write a valid Snakemake rule with `input`, `output`, and `shell`.
- Execute a specific target file.

::::::::::::::::::::::::::::::::::::::::::::::::

![Snakemake Logo](https://raw.githubusercontent.com/snakemake/snakemake/main/docs/logo-snake.svg){.img-responsive .float-right style="width: 150px; margin-left: 20px;"}

## Why Snakemake for CMS?

In CMS analysis, we rarely run a single script. We create an analysis chain: **Skimming $\rightarrow$ Processing $\rightarrow$ Plotting $\rightarrow$ Fitting.** Traditionally, we managed this with "Mega-Bash" scripts or by manually submitting JDL files to HTCondor/Slurm.

### Key Advantages

* **Portability:** Works on your laptop, LXPLUS, or the LPC with the same code.
* **Environment Management:** Seamless integration with **Containers** (Apptainer/Singularity) and Conda.
* **Visualization:** It automatically generates a "map" (Directed Acyclic Graph or DAG) of your analysis.

### Snakemake vs. LAW (Luigi Analysis Workflow): A bias comparison

* **Boilerplate:** LAW is based on Luigi and requires significant Python boilerplate for every task. Snakemake uses a "Domain Specific Language" (DSL) that is much closer to plain English or a configuration file.
* **The Learning Curve:** Snakemake is widely used in Genomics and Data Science. If you search for a problem on StackOverflow, you'll find an answer in seconds. LAW is more niche to the HEP community.
* **Automation:** Snakemake is **file-based**. It looks at the "last modified" timestamp of your files. If you change your plotting script, it won't re-run the 5-hour skimming step unless you ask it to.
* **Workflow Isolation:** All your code can run independently of Snakemake. You can think of Snakemake as a "workflow manager" that orchestrates your existing scripts (like a bash script on steroids). LAW requires you to write your tasks as Luigi Tasks, which can be more cumbersome.

:::: callout

### Credits and Documentation

Snakemake is an open-source project created by **Johannes Köster** (University of Duisburg-Essen) in 2012. While this tutorial focuses on CMS-specific use cases, the official documentation is comprehensive and covers thousands of features.

*   **Documentation**: [https://snakemake.github.io](https://snakemake.github.io)
*   **Citation**: If you use Snakemake in your analysis, proper citation is expected in our field:

> Köster, Johannes and Rahmann, Sven. "Snakemake—a scalable bioinformatics workflow engine". *Bioinformatics*, 2012.

::::::::::::::::

---

## The Essentials

To run a workflow, you need two things:

1. **The Snakefile**: A file named `Snakefile` (capital S, no extension) where you define your rules. It uses a Python-based syntax, so you can write standard Python code inside it alongside your rules.
2. **The Execution Command**: You run the workflow from the terminal by invoking `snakemake`.

The basic command structure is:

```bash
snakemake --cores <N> <target_file>
```

*  `--cores <N>`: Tells Snakemake how many CPU cores to use. (e.g., `--cores 1` for sequential execution, `--cores 4` for parallel).
*  `<target_file>`: The file you want to generate. Snakemake will figure out the steps to get there.

---

## The Anatomy of a Rule

The building block of Snakemake is the **rule**. Think of it as a recipe. To make a dish, you need ingredients (inputs), a kitchen (the environment), and a set of instructions (the shell command).

```python
rule skim_data:
    input:
        "raw_data.txt"
    output:
        "skimmed_data.txt"
    shell:
        "grep 'Signal' {input} > {output}"
```

Important Properties:

* **Rule Name**: Must be unique.
* **Input/Output**: These are strings (or lists of strings). Snakemake uses these to "connect the dots" between rules.
* **Shell**: The bash command to execute. Notice the `{input}` and `{output}` placeholders—Snakemake automatically fills these in with the paths you defined above.

### Understanding the Syntax

A `Snakefile` is essentially a Python script with some extra keywords added. This is powerful because it means you can use Python variables, functions, and libraries directly in your workflow definition.

* **Indentation matters**: Just like in Python, Snakemake uses indentation (usually 4 spaces) to group code blocks. The `input`, `output`, and `shell` directives must be indented relative to the `rule`.
* **Strings**: File paths and commands are strings, so they must be enclosed in quotes (`"..."` or `'...'`).
* **Comments**: Use `#` for comments, just like in Python/Bash.
* **Lists**: If a rule has multiple inputs or outputs, you can list them with commas:

```python
    input:
        "file1.txt",
        "file2.txt"
```

## Running the Rule

1. Create dummy data:

```bash
echo -e "Background\nSignal\nBackground\nSignal" > raw_data.txt
```

2. Run Snakemake. We must tell it **what file we want to generate**:

```bash
pixi run snakemake --cores 1 skimmed_data.txt
# if you run snakemake from a conda environment or pip, use:
# snakemake --cores 1 skimmed_data.txt
```

If successful, you will see `Finished jobid: 0`.

:::: spoiler

### Output

```output
Assuming unrestricted shared filesystem usage.
host: gluon
Building DAG of jobs...
Using shell: /usr/bin/bash
Provided cores: 1 (use --cores to define parallelism)
Rules claiming more threads will be scaled down.
Job stats:
job          count
---------  -------
skim_data        1
total            1

Select jobs to execute...
Execute 1 jobs...

[Sun Feb  8 12:05:08 2026]
localrule skim_data:
    input: raw_data.txt
    output: skimmed_data.txt
    jobid: 0
    reason: Missing output files: skimmed_data.txt
    resources: tmpdir=/tmp
[Sun Feb  8 12:05:08 2026]
Finished jobid: 0 (Rule: skim_data)
1 of 1 steps (100%) done
```

::::::::::::::::::::::::

:::: instructor

If `snakemake` was not installed using pixi but with a conda environment or pip, you should remove the `pixi run` part and just run:

```bash
snakemake --cores 1 counts.txt
```

This will be the case for most users following this tutorial outside of the pixi environment.

:::::::::

:::: challenge

### Syntax Error Hunt

Intentionally break your indentation (remove a space before `input:`). Run the command again.

What error does Snakemake give you?

:::: solution

This `IndentationError` is the most common error you will encounter.

::::::::
::::::::

::::::::::::::::::::::::::::::::::::: keypoints

* A `Snakefile` defines the workflow.
* A `rule` contains `input`, `output`, and a `shell` command.
* You execute the workflow by asking for the **output file**, not the rule name.

::::::::::::::::::::::::::::::::::::::::::::::::
