# PROJECT PROPOSAL: Statistical Analysis of Effects of Weather Conditions on Consumer Behavior and Return Rates

Prepared by: **Kaan Bayri**

## 1. Project Objective and Problem Statement
Weather condition is known to effect human psychology for a long time, yet the impact on online shopping is rarely analyzed through data. This project aims to analyze the causal effect of different weather conditions (rainy, sunny, cold, hot) on e-commerce purchasing volume, average order value, and return/cancellation rates.
My core research question is whether bad weather triggers an increase in boredom induced, higher budget, and impulsive purchases, and whether these specific orders have a higher tendency to be canceled or returned.

## 2. Data Souces
The project will integrate two distinct structural datasets:
* **Olist Brazilian E-Commerce Dataset:** A relational database containing order dates, cart values, customer locations (coordinate based), and order statuses (delivered/canceled) between 2016 and 2018.
* **INMET Automatic Weather Stations Data:** Time series data containing daily temperature, precipitation, and geographical coordinates of meteorological stations in Brazil for the same time period.

## 3. Methodology and Technical Approach
The project consists of two main pillars: data engineering and statistical analysis.
* **Spatial Mapping:** Customer geographical coordinates and weather station coordinates will be mapped using distance algorithms such as the *Haversine* formula, linking each order transaction to the nearest meteorological data point.
* **Relational Data Architecture:** Order and return/cancellation dates will be merged with the external weather dataset. This will allow us to determine the weather conditions at the "time of purchase" for each specific transaction.
* **Statistical Testing:** Grouping analyses will be performed on the merged dataset to evaluate the impact of weather on average cart value and return rates in terms of statistical significance.

## 4. Expected Outcomes
This study aims to provide concrete, data-driven insights that can help e-commerce companies develop dynamic, weather-based targeted marketing strategies and plan their return logistics more efficiently.
