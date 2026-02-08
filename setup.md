---
title: Setup
---

## Basic Knowledge

This tutorial assumes you have a basic understanding of the following packages and concepts:

* **git**: Version control system for tracking changes in code.
* **containers**: Lightweight, portable environments for running applications (e.g., Docker, Singularity).
* **conda**: Package and environment management system.
* **coffea**: A Python library for high-energy physics data analysis. (You won't program anything in coffea during this tutorial, but it's good to know what it is.)

## Setup and Installation

Most of this tutorial can be run on your local machine, but we will also demonstrate how to run a heavy-duty physics workflow on a remote cluster.

We will prepare our local environment using **Pixi** to manage Snakemake. While the heavy-duty physics code will eventually run inside CMSSW or conda environments, or containers, we need a reliable "Orchestrator" on our laptops to manage the workflow.

::::::::::::::: discussion

## Why Pixi instead of Conda?

For years, `conda` (and `mamba`) has been the standard for HEP environment management. However, **Pixi** is a modern alternative built on the same foundations (Conda-forge) but with several key advantages:

* **Speed:** It is significantly faster at resolving dependencies than standard Conda.
* **Reproducibility:** It creates a `pixi.lock` file automatically, ensuring that every student in this tutorial has the *exact* same version of every package.
* **Project-Centric:** Pixi keeps dependencies local to your project folder rather than burying them in a global `/envs/` directory.
* **Single Tool:** It handles environment creation, package installation, and task execution (like `make`) in one binary.

More information about Pixi can be found in the [Pixi documentation](https://pixi.sh/docs/).

:::::::::::::::::::::::::

---

### 1. Installing Pixi

First, we need to install the `pixi` binary itself. Open your terminal and run the command appropriate for your system:

**macOS and Linux:**

```bash
curl -fsSL https://pixi.sh/install.sh | bash
```

(Note: You may need to restart your terminal or source your .bashrc / .zshrc after installation.)

Verify the installation:

```bash
pixi --version
```

### 2. Creating the Project and Installing Snakemake

Now, we will initialize a new project directory and install Snakemake. This ensures that our workflow is self-contained.

```bash

# 1. Create a new directory for the tutorial
mkdir snakemake-cms-tutorial
cd snakemake-cms-tutorial

# 2. Initialize a pixi project
pixi init .

# 3. Add Snakemake and Graphviz
# (Graphviz is used to visualize our workflow diagrams)
pixi add snakemake graphviz
```

### 3. Verification

To ensure everything is working correctly, we will run a simple command through the pixi environment. Pixi uses the run command to execute software inside the environment it just created.

```bash
pixi run snakemake --version
```

If you see a version number (e.g., `8.x.x`), you are ready to go!

:::: challenge

### Check your environment

Look inside your cms-snakemake-workshop folder. Can you find the pixi.toml file? Open it with a text editor and identify where snakemake is listed.

::::::::::

:::: spoiler

### If you want to use conda instead...

If you prefer to use `conda` instead of `pixi`, you can create a conda environment and install Snakemake there. However, keep in mind that this tutorial is designed around Pixi's workflow management, so you may need to adjust some commands accordingly.

```bash

## Create the environment
conda create -c conda-forge -c bioconda -n snakemake snakemake

# Activate the environment:
conda activate snakemake

# Verify the installation:
snakemake --version
```
::::::::::