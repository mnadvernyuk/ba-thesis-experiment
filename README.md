# ⛓️ Causal Attribution in Causal Chains

This repository contains all materials, data, and analysis scripts for my bachelor’s thesis:

**“Causal Attribution in Causal Chains”**

completed at the **University of Tübingen** under the supervision of  
**Prof. Dr. Michael Franke**.

## 📄 Overview

This thesis investigates how people select causes from unfolding causal chains, focusing on two central factors in causal cognition:

- **Distal type of an event**  
  (natural vs. non-deliberate vs. deliberate)
- **Outcome valence framing**  
  (positive vs. negative outcomes)

Building on earlier work on causal selection in event chains (e.g., Hilton & McClure, 2009), the study examines whether people are more likely to identify **intentional or agentive events** as “the cause,” even when these events are **not temporally closest** to the outcome.

The experiment uses short vignette-based causal chains in which participants select the event they consider the best candidate for causing the outcome.

## 🧪 Experiment

Participants completed an online causal selection task in which they read a series of short stories describing causal chains followed by an outcome.

On each trial, participants chose which event best caused the outcome:

- **Event A** – precondition / enabling event  
- **Event B** – distal event  
- **Event C** – proximal event  
- **Event D** – immediate event  

Each scenario was presented with either a **positive** or **negative** outcome, allowing an exploratory test of valence framing effects.

The experiment was implemented using **Vue** and **magpie**.  
Data collection was conducted via **Prolific**, with an additional pilot sample recruited informally among friends.

🔗 **Live experiment:**  
👉 https://mnadvernyuk.github.io/ba-thesis-experiment/

## 📊 Repository Structure

## 🧮 Data Analysis

Data were analyzed in **R** using descriptive statistics and inferential tests inspired by prior work on causal chain selection.

The primary dependent variable was **distal selection** (whether participants selected Event B).

Analyses compare distal selection rates across:

- distal event types (natural / non-deliberate / deliberate)
- outcome valence (positive vs. negative)
- datasets including all participants vs. excluding attention-check failures

All analysis scripts are located in the `analysis/` directory.

## 📌 Main Findings (Summary)

- Participants showed a consistent **descriptive tendency** to select the distal event more often when it was **deliberate** than when it was natural or non-deliberate.
- This pattern remained visible after excluding participants who failed the attention check.
- **Outcome valence framing** produced only small descriptive differences and no strong inferential effects.
- Overall, the results suggest that **agency and intentionality** play a central role in causal attribution within unfolding causal chains.

## 📄 Thesis

A PDF of the completed thesis:

**“Causal Attribution in Causal Chains”**

## 💬 Contact

For questions, comments, or collaboration inquiries, feel free to contact me:

📧 **Email:** mnadveriyk@gmail.com  
📓 **GitHub:** https://github.com/mnadvernyuk



