---
title: "The Why: Analysis Reproducibility"
teaching: 10
exercises: 0
---

:::::::::::::::::::::::::::::::::::::: questions 

- Why do we need workflow orchestration in CMS?
- What are the three pillars of a reusable analysis?
- Who is the primary beneficiary of a reproducible workflow?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Identify the "Knowledge Gap" in traditional HEP analyses.
- Understand the three-step approach to capturing an analysis.
- Recognize that reproducibility is a labor-saving tool for the analyst, not just a bureaucratic requirement.

::::::::::::::::::::::::::::::::::::::::::::::::

## The "Bus Factor" in CMS Analysis

In a typical CMS analysis, we move through **Data Exploration**, **Background Modeling**, **Systematics**, and **Statistical Fits**. Usually, this is handled by a series of disconnected scripts and manual steps.

The problem arises when an analyst moves on or a "Future You" returns to the code six months later:

* **The Environment:** What version of CMSSW or Coffea was used?
* **The Logic:** Why was this specific histogram used as input for that fit?
* **The Commands:** What were the exact arguments passed to the script?

Without a workflow manager, this knowledge is lost, leading to "Scientific Archeology" where we try to dig up our own results.

## The Three Pillars of Reusability

To solve this, we think of an analysis in three layers of "Capturing":

1. **Capture Code & Environment:** Use **Git** for code and **Containers** (Docker/Apptainer) or **Pixi/Conda** for the software. This ensures the "Kitchen" is always the same.
2. **Capture Commands:** Move away from manual terminal commands. Use a script or a rule that explicitly states how to run the code.
3. **Capture Workflow:** This is the "Glue." We define the **relationship** between steps. This is where **Snakemake** comes in.



## Why should you care?

You might think: *"Reproducibility is for the collaboration, but I'm just trying to graduate."*

Actually, the first person to benefit from a reproducible workflow is **you**. 

* **The "Re-discovery" Phase:** During the CMS review process (ARC), you will inevitably be asked to change a systematic or re-run a plot with new data. A Snakemake workflow allows you to do this by changing one line and typing one command.
* **Automation:** Instead of waiting for Step A to finish so you can manually start Step B, you can launch the whole chain and go get coffee.

:::::::::::::::::::::::::::::::::::::::::::::::::::::: challenge

## Who is the most frequent user of your analysis code?

:::::::::::::::::::::: solution
**Future You.** Six months from now, when you have to re-run your plots for a conference or a paper revision, you will thank "Past You" for writing a Snakefile instead of a 20-page README.
::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::: keypoints

- Modern CMS analysis is too complex to be managed by memory or manual scripts.
- Reproducibility is a **productivity tool**: it makes your own work easier to revise and update.
- Capturing the **Workflow** (the "Glue") is the final step in ensuring an analysis is truly reusable.

::::::::::::::::::::::::::::::::::::::::::::::::