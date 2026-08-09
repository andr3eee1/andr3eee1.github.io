---
layout: page
title: Projects
icon: fas fa-code-branch
order: 1
---

## QuantOptima
*Independent Research Project on Neural Network Codebooks*

An investigation into whether standard quantization formats (INT8, FP8) are optimally designed, or if calculating a custom dynamic codebook using the Lloyd-Max algorithm yields better accuracy. 

* **Key Finding:** Proved the "MSE Paradox"—minimizing theoretical rounding error actually destroys real-world model accuracy by smoothing out critical weight outliers.
* **Deliverables:** Custom bitwise PyTorch encoders, 1D k-means optimization engines, synthetic kurtosis crossover analysis, and a comprehensive Typst academic report.

👉 [View Repository on GitHub](https://github.com/andr3eee1/QuantOptima)