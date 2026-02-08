---
title: 'Containerized Execution'
teaching: 15
exercises: 15
---

:::::::::::::::::::::::::::::::::::::: questions

- How do I run specific steps of my analysis in a controlled environment?
- How can I use CMSSW or specific Python versions without installing them locally?
- How does Snakemake handle Apptainer/Singularity?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Use the `container:` directive to link a rule to a Docker/Apptainer image.
- Execute a workflow where different rules use different environments.
- Understand the `--use-apptainer` (or `--use-singularity`) flag.

::::::::::::::::::::::::::::::::::::::::::::::::

## Containers: Your Analysis in a Box

In CMS, we often need very specific environments: a certain version of CMSSW, a specific ROOT version for `combine`, or a set of Python libraries like `coffea`. Instead of spending hours fighting with `export PATH` or `cmsenv`, we can use **Containers**.

Snakemake makes this seamless. You can tell a specific rule to run inside a container, and Snakemake will automatically pull the image and wrap your command inside it.

---

:::: instructor

### My Opinions on this Episode

* **The "LPC/LXPLUS" connection:** This is where you should mention that on most HEP clusters, `singularity` or `apptainer` is already installed. This makes their local tutorial 100% transferable to the big machines.
* **Binding directories:** Students often ask how the container sees their files. It's worth a small note that Snakemake automatically "binds" the project directory so the container sees the code and data.

:::::::::::::::::::::::::::

## The `container:` Directive

To use a container, you simply add the `container:` keyword to your rule.

```python
rule plot_data:
    input:
        "results/{dataset}_counts.txt"
    output:
        "plots/{dataset}.png"
    container:
        "docker://python:3.10-slim"
    shell:
        "python scripts/my_plotter.py {input} {output}"
```

### What happens behind the scenes?

When you run Snakemake with the `--use-apptainer` flag:

1. Snakemake sees the `container:` directive.
2. It pulls the image (if not already present) using Apptainer/Singularity. *Note*: Apptainer can run `docker://` images perfectly fine.
3. It starts the container and **automatically mounts (binds)** your current working directory inside it.
4. It executes the `shell` command inside that container.

### Why is this better than a local environment?

- **Portability**: You can run the exact same container on your laptop, the LPC, or the Grid.
- **Isolation**: Rule A can use `python:2.7` while Rule B uses `python:3.11`. *No more version conflicts!*
- **No Installation**: You don't need to install `ROOT` or `CMSSW` on your machine; you just need to point to the image.

:::: callout

### A Note on Apptainer Installation

While we are using **Pixi** to manage Snakemake and our local Python tools, Pixi does not typically install Apptainer/Singularity itself. This is because Apptainer requires specific system-level permissions to manage containers safely. 

* **On your laptop:** You must have Apptainer installed at the system level (e.g., via `brew` on macOS with a virtual machine, or your Linux distribution's package manager). 
* **On CMS Clusters (LPC/LXPLUS):** Apptainer is already pre-installed by the administrators. 

Before proceeding, verify you have it by running: `apptainer --version`.

::::::::::::::::

---

## Activity: Running a "Physics" Script in a Container

Let's simulate a plotting step that requires a specific Python environment.

1. Create a simple plotting script named `plotter.py`:

```python
import sys
# Simulate a plotting library requirement
print(f"Generating plot from {sys.argv[1]} using Python {sys.version}")
with open(sys.argv[2], "w") as f:
    f.write("IMAGE_DATA")
```

2. Modify your `Snakefile` to include a containerized plotting rule:

```python
DATASETS = ["DYJets", "TTbar", "Data"]

rule all:
    input:
        expand("plots/{d}.png", d=DATASETS)

# (Keep your previous skim_data and count_events rules here)

rule plot_results:
    input:
        "results/{dataset}_counts.txt"
    output:
        "plots/{dataset}.png"
    container:
        "docker://python:3.10-slim"
    shell:
        "python plotter.py {input} {output}"
```

3. Run the workflow using Apptainer:

```bash
pixi run snakemake --cores 1 --use-apptainer
```

:::: callout

Please note that the previous exercise will create empty "plots" since the `plotter.py` is just a placeholder. The point is to see the container in action, not to generate real plots!

:::::::::::::::::::::::::::

**Did it work?** Depending on where you run it, this answer may vary. If you are running on `lxplus`/`cmslpc`, you might get an error about bindings or permissions. This is the next topic we'll cover.

## Accessing External Data (Bind Mounts)

By default, Snakemake only lets the container see files inside your current project folder.

**The CMS Problem:** In HEP, our data typically lives on storage areas like `/eos` or `/cernbox`, and our software might live on `/cvmfs`. If you try to access a file in `/eos/user/...` from inside the container, it will fail because the container is isolated.

**The Solution:** You can pass arguments to Apptainer using the `--apptainer-args` flag in Snakemake:

```bash
pixi run snakemake --cores 1 --use-apptainer --apptainer-args "--bind /eos:/eos --bind /cvmfs:/cvmfs"  ### --bind /uscms_data/d3/user/ if you are at the LPC
```

This tells Apptainer: "Poke a hole in the container so I can see `/eos` and `/cvmfs` from the outside."

:::: challenge

### Exercise: Different Containers for Different Tasks

Imagine your `skim_data` rule requires an old C++ library only available in a `cmssw` image, but your `plot_results` rule needs a modern `coffea` environment.

1. Can you assign different `container:` directives to different rules in the same `Snakefile`?

2. Try changing the container: in `plot_results` to `docker://alpine:latest` and run it. What happens?

:::: solution 

Yes! Snakemake is designed for this. It will start the correct container for each specific job. If you switch to alpine:latest, the job will fail because alpine does not have python installed by default—this proves the command is truly running inside the isolated container!
::::::::::::::::

::::::::::::::::

:::::::::::::::::::::::::::::::::::: keypoints

- **container:**: A rule-level directive that specifies the Docker/Apptainer image to use.

- **--use-apptainer**: The command-line flag required to enable container execution.

- ***--apptainer-args**: Use this to bind external storage paths (like `/eos` or `/cvmfs`) so the container can see them.

- **Environment Agnostic**: You can mix and match different containers in a single workflow, ensuring each step has the exact dependencies it needs.

::::::::::::::::::::::::::::::::::::::::::::::::
