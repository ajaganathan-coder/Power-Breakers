---
layout: default
title: Power Breakers
---

# Power Breakers - DSC259R Final Project

**Names**:

1. Mitchell Farrington mfarring@ucsd.edu - A15378625
2. Anuradha Jaganathan ajaganathan@ucsd.edu - A69038119
3. Kyle Packer kpacker@ucsd.edu - A69036932
4. Alex Twoy atwoy@ucsd.edu - A17580384


**Website Link**: [https://ajaganathan-coder.github.io/Power-Breakers/](https://ajaganathan-coder.github.io/Power-Breakers/)  
---

## Introduction

Prolonged power outages during summer and winter months pose significant risks, particularly when they exceed 24 hours(>1440 mins). These disruptions can trigger serious health crises for populations in affected regions. Leveraging historical outage data to predict the occurrence of major events can enable cities to prepare proactively and minimize potential impacts. We discussed on the below questions 

  1. Which areas suffered the most power outages and why?
  2. What was the average length of power outages?
  3. Does urban vs. rural area type affect outage frequency?                                                            
  4. Can we predict whether a major severe-weather outage will occur in a NERC region during a given week?

and finally decided on our research ### Research Question
> **For each NERC region r and week w, estimate the probability that at least one major outage triggered by severe weather will occur in region r during week w.?**

Answering this question has direct operational value and help cities and repair crews to prepare ahead of time.

### Dataset
The **Purdue Major Power Outage Risk dataset** [https://engineering.purdue.edu/LASCI/research-data/outages](https://engineering.purdue.edu/LASCI/research-data/outages), which records 1,534 documented outage events across the contiguous United States from January 2000 to July 2016 was chosen for the project. Each event describes one outage event with 55 attributes spanning:
- **Temporal context** — year, month, start/restore timestamps
- **Geographic identifiers** — state, NERC region, climate region
- **Meteorological context** — climate category (Normal / El Niño / La Niña),   anomaly level
- **Cause information** — broad cause category and detailed descriptor
- **Socioeconomic covariates** — population, urban fraction, GSP per capita
- **Outage outcome** — duration (minutes), customers affected, MW demand lost

The columns relevant to our analysis are:

## Data Cleaning and Exploratory Data Analysis

To prepare the data for exploratory and further analysis, the below data cleansing steps were performed:

### Drop irrelevant columns
The raw workbook includes price/sales columns (retail electricity tariffs and consumption by sector) and GSP/utility columns that describe the state's economic profile *in aggregate* rather than the outage event. These are redundant given we already have `PC_REALGSP_*` aggregates and would introduce multicollinearity. We also drop the Purdue internal `variables` descriptor column.

### Rename columns
Dots in column names make attribute access fragile. We replace every `.` with `_`.

### Merge date + time → Timestamps
Outage start and restoration are stored as separate date and clock-time columns. We concatenate them into proper `pd.Timestamp` objects so we can compute arithmetic like `OUTAGE_DURATION_MINS = (restore - start).total_seconds() / 60`.

### Cast integer columns
Several columns arrive as float because of NaNs. After cleaning we downcast where the column only ever holds whole numbers.

### Drop rows with missing `OUTAGE_DURATION_MINS`
Rows where the duration is unknown cannot be labelled and are excluded. This affects 58 of 1 534 rows (3.8 %).

We then addressed below questions as part of the exploratoaty analysis to understand more about the data and filter to the required content.
*Key questions:*
- Is outage duration right-skewed? (Almost certainly yes — most outages are short, but a tail of catastrophic events pulls the mean up.)
<iframe src="iframe_figures/figure_9.html" width=800 height=600 frameBorder=0></iframe>

- Which states / NERC regions suffer the most events or the longest outages?
<iframe src="iframe_figures/figure_9.html" width=800 height=600 frameBorder=0></iframe>

- Is there a seasonal signal in duration?
<iframe src="iframe_figures/figure_9.html" width=800 height=600 frameBorder=0></iframe>

- How correlated are the numeric features?
<iframe src="iframe_figures/figure_9.html" width=800 height=600 frameBorder=0></iframe>

- What fraction of outages cross the 1 440-minute threshold we will predict?


