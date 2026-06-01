# PAARL: An Interactive System for Policy-Aware Planning in Autonomous Agents

## Overview

PAARL (Policy-Aware Autonomous Reasoning and Learning) is an interactive framework for policy-aware planning in autonomous agents operating in dynamic and norm-governed environments.

The system extends the Observe–Diagnose–Plan–Execute (ODPE) agent architecture by integrating:

- Diagnostic reasoning
- Policy-aware planning
- Penalty assessment
- Incremental replanning
- Explainable decision making

PAARL combines Answer Set Programming (ASP), the Clingo solver, and a Python-based graphical user interface to provide an interactive environment for simulating autonomous agent behavior under normative constraints.

This repository contains the reference implementation accompanying the paper:

**PAARL: An Interactive System for Policy-Aware Planning in Autonomous Agents**

---

## Quick Start

```bash
git clone https://github.com/vineelsai313/PAARL.git
cd PAARL

pip install -r requirements.txt

python paarl.py
```

---

# System Architecture

PAARL consists of the following major components.

## Graphical User Interface (GUI)

The GUI provides an interactive environment for:

- Creating simulation scenarios
- Defining goals
- Specifying emergency and non-emergency situations
- Injecting observations during execution
- Running diagnosis and planning modules
- Visualizing outputs and execution history

## Python Controller

The Python controller manages the entire agent loop by:

- Generating ASP scenario files
- Invoking Clingo
- Parsing answer sets
- Managing trajectories
- Coordinating diagnosis and planning
- Maintaining simulation state across time steps

## Diagnosis Module

The diagnosis component explains unexpected observations through abductive reasoning and identifies exogenous actions that may have occurred in the environment.

## Planning Module

The planning component generates policy-aware plans that achieve agent goals while considering:

- Traffic regulations
- Normative constraints
- Penalties
- Execution costs
- Emergency priorities

## Persistent Trajectory Manager

Maintains the complete execution history and automatically propagates relevant actions and observations across future reasoning cycles.

---

# Repository Structure

```text
PAARL/
│
├── README.md
├── requirements.txt
├── paarl.py
│
├── applicable_policies.txt
├── diagnosis.txt
├── dynamic_domain.txt
├── dynamic_domain_planning.txt
├── find_loc.txt
├── penalty_and_time.txt
├── planning.txt
├── policies.txt
├── reality_check.txt
│
└── runs/
    ├── scenario_YYYY-MM-DD_HH-MM-SS_startX_goalY_mode/
        ├── inputs/
        ├── outputs/
        └── logs/
```

---

# ASP Components

## dynamic_domain.txt

Contains:

- Action definitions
- Fluent definitions
- State transition rules
- Inertia axioms
- Domain dynamics

## dynamic_domain_planning.txt

Contains planning-specific domain rules and auxiliary reasoning components used by the planning module.

## policies.txt

Contains:

- Traffic regulations
- Policy definitions
- Normative constraints
- Emergency policy exceptions

## planning.txt

Contains ASP planning rules used to generate plans.

## diagnosis.txt

Contains diagnostic reasoning rules used to explain unexpected observations.

## reality_check.txt

Contains consistency constraints and reality verification rules.

## applicable_policies.txt

Contains rules used to determine which policies are applicable in a given state and scenario.

## find_loc.txt

Contains helper rules used for location reasoning.

## penalty_and_time.txt

Contains:

- Penalty calculations
- Priority assignments
- Execution time calculations
- Cost aggregation rules

---

# Requirements

## Python

Python 3.10 or later is recommended.

Verify installation:

```bash
python --version
```

## Python Dependencies

Install the required GUI dependencies:

```bash
pip install -r requirements.txt
```

Current requirements:

```txt
PyQt5>=5.15.0
```

## Clingo

PAARL uses Clingo for Answer Set Programming reasoning.

### Windows

Download Clingo from:

https://github.com/potassco/clingo/releases

Extract the archive and add the Clingo executable to your system PATH.

Verify installation:

```bash
clingo --version
```

### Ubuntu/Linux

```bash
sudo apt install clingo
```

### macOS

```bash
brew install clingo
```

---

# Installation

## Clone Repository

```bash
git clone https://github.com/vineelsai313/PAARL.git
cd PAARL
```

## Install Python Dependencies

```bash
pip install -r requirements.txt
```

## Install Clingo

Download and install Clingo from:

https://github.com/potassco/clingo/releases

Verify installation:

```bash
clingo --version
```

---

# Running PAARL

Launch the GUI:

```bash
python paarl.py
```

---

# Agent Loop Workflow

PAARL implements an iterative Observe–Diagnose–Plan–Execute cycle.

## Step 0: Initialize Scenario

The user specifies:

- Initial location
- Goal location
- Emergency or non-emergency status
- Initial observations

PAARL generates:

```text
scenario_step_0_planning.txt
```

and executes:

```bash
clingo dynamic_domain.txt dynamic_domain_planning.txt policies.txt planning.txt reality_check.txt applicable_policies.txt penalty_and_time.txt scenario_step_0_planning.txt
```

The generated plan is stored in:

```text
output_step_0_planning.txt
```

---

## Step 1: Observe

After execution, the user may introduce new observations.

Example:

```prolog
obs(at_loc(4),true,1).
obs(light_color(red,3,4),true,1).
```

---

## Step 2: Diagnose

PAARL generates:

```text
scenario_step_1_diagnosis.txt
```

and performs diagnostic reasoning.

Example output:

```prolog
expl(change_light(3,4),1)
```

The explanation is incorporated into future planning.

---

## Step 3: Replan

PAARL generates:

```text
scenario_step_1_planning.txt
```

and computes a new policy-aware plan using:

- Current observations
- Diagnostic explanations
- Previous execution history
- Applicable policies

---

## Step 4: Execute

The selected action is executed and the cycle repeats until:

- The goal is achieved
- No valid plan exists
- The user terminates the simulation

---

# Generated Files

PAARL automatically creates a dedicated run folder for each scenario execution.

Example:

```text
runs/
└── scenario_2026-01-15_14-30-00_start5_goal10_emergency/
    ├── inputs/
    ├── outputs/
    └── logs/
```

## Inputs

Contains generated scenario files:

```text
scenario_step_0_planning.txt
scenario_step_1_diagnosis.txt
scenario_step_1_planning.txt
scenario_step_2_diagnosis.txt
scenario_step_2_planning.txt
...
```

## Outputs

Contains planning and diagnosis summaries:

```text
output_step_0_planning.txt
output_step_1_diagnosis.txt
output_step_1_planning.txt
...
```

## Logs

Contains raw Clingo execution logs and debugging information for each step.

---

# Features

## Interactive Planning

Users may inject observations at any time step without restarting the simulation.

## Policy-Aware Reasoning

Plans are generated while considering:

- Norm compliance
- Policy violations
- Penalties
- Execution costs

## Diagnostic Reasoning

Unexpected observations trigger diagnosis to explain environmental changes.

## Incremental Replanning

The agent continuously adapts plans as new information becomes available.

## Explainable Decisions

Generated plans include detailed reasoning traces from ASP answer sets.

## Emergency Reasoning

Emergency situations may justify policy violations when required to achieve critical goals.

## Scenario Session Management

Each simulation run is stored in an isolated folder containing:

- Generated inputs
- Planning outputs
- Diagnosis outputs
- Raw Clingo logs

This allows experiments to be reproduced and analyzed after execution.

---

# Example Use Cases

- Autonomous vehicle navigation
- Emergency response planning
- Norm-governed autonomous systems
- Policy-aware robotics
- Explainable AI planning systems
- Intelligent transportation research

---

# Research Context

This repository contains the implementation used in:

**PAARL: An Interactive System for Policy-Aware Planning in Autonomous Agents**

The framework demonstrates how Answer Set Programming can be used to support policy-aware reasoning, diagnosis, planning, execution, and replanning in autonomous agents operating within dynamic environments governed by norms and policies.

# License

This project is released for research and educational purposes.
