---
layout: post
title: "QuantOptima: Why the 'Perfect' Math Fails in Neural Network Quantization"
date: 2026-08-10 12:00:00 +0300
categories: [Machine Learning, Systems]
tags: [quantization, python, pytorch, research, lloyd-max]
pin: true
---

Large Language Models and deep computer vision architectures use billions of parameters. Storing them in standard 32-bit floating point takes up too much memory space, making them impossible to run on normal hardware. To fix this, I compressed the memory arrays to 8-bit or 4-bit representations. 

This is the journey of **QuantOptima**, my independent research project investigating whether standard industry formats (like INT8 and FP8) are optimally designed, or if calculating a custom, dynamic codebook using a Lloyd-Max k-means algorithm provides better accuracy. 

Along the way, I discovered a major paradox: minimizing the theoretical rounding error actually destroys real model accuracy because it ignores critical outliers in the weight distribution.

---

## 1. The Memory Wall

When a neural network is trained, it learns patterns by adjusting the values of its weights. In frameworks like PyTorch, these weights are stored as `float32` variables, where a single float takes 4 bytes of memory. 

If you take a model like GPT-3 with 175 billion parameters, just storing the weights requires about 700 Gigabytes of VRAM. A high-end consumer graphics card like the RTX 4090 only has 24GB. This massive gap is why quantization is not just a fun optimization trick, but a strictly mandatory step for running modern AI.

Quantization means forcing those continuous 32-bit decimals into a smaller bucket of discrete values. If I use 8 bits, I only have $2^8 = 256$ unique values to represent my numbers. If I drop to 4 bits, I only get $2^4 = 16$ unique values. 

The core question of this project was simple: **If I only have 16 slots, what are the exact mathematically perfect values I should pick to fill them?**

---

## 2. Training the Baseline & The Standard Formats (Phase 1)

Before compressing anything, I needed a real model with real weights. I built a Convolutional Neural Network (CNN) from scratch in PyTorch to classify images from the CIFAR-10 dataset, achieving a baseline accuracy of **75.05%** in full FP32 precision.

To understand what I was trying to compress, I extracted the raw memory tensors from the trained model and plotted their histograms. 

![Weight distributions for conv1 and fc1 layers](/assets/img/conv1_weight.png)

The distributions naturally clump really tight around zero, resembling a standard bell curve. However, tiny tails stretch outwards to the extremes on the X-axis. These rare extremes are the **outliers**, and they make quantization extremely difficult.

To see how the industry handles this, I wrote custom encoders and decoders from scratch for four major formats:
- **INT8 (Integer 8-bit):** Uses a rigid, evenly spaced symmetric grid ($Q(x) = \lfloor x / s \rfloor \cdot s$).
- **FP8 (E4M3):** Uses a 1-bit sign, 4-bit exponent, and 3-bit mantissa to cluster values tightly near zero.
- **Posit8:** Uses dynamic regime bits to allocate precision where it matters most.
- **NF4 (NormalFloat 4-bit):** Assumes weights follow a Gaussian bell curve and hardcodes exact theoretical quantiles.

Surprisingly, when I benchmarked them, **INT8 scored 75.07% (a drop of -0.02%)**. The negative drop happens because the rigid quantization grid acts as a regularizer, filtering out the overfitting noise from the training data!

---

## 3. Building the "Perfect" Codebook (Phase 2)

Instead of guessing the shape of the weights like standard formats do, I wrote a Lloyd-Max algorithm to find the theoretically perfect 1D codebook for every single layer independently. This is essentially a 1D k-means clustering algorithm designed to strictly minimize the Mean Squared Error (MSE) between the original weights $W$ and the quantized weights $Q(W)$:

$$\text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (W_i - Q(W_i))^2$$

To prove why moving the centroid to the mean minimizes the error, I derived it using pure algebra (avoiding calculus limits, as preferred for elementary derivations):

Let $S_k$ be the set of weights assigned to a cluster, $C$ be the centroid, and $\mu$ be the true arithmetic mean of the cluster:
$$\mu = \frac{\sum_{x \in S_k} x}{|S_k|}$$

Expanding the squared error for the cluster:
$$\text{Error} = \sum_{x \in S_k} (x - C)^2$$

Adding and subtracting $\mu$ inside the square:
$$\text{Error} = \sum_{x \in S_k} (x - \mu + \mu - C)^2$$

Expanding the quadratic expression:
$$\text{Error} = \sum_{x \in S_k} \left[ (x - \mu)^2 + 2(x - \mu)(\mu - C) + (\mu - C)^2 \right]$$

Since the sum of differences from the mean ($\sum (x - \mu)$) is always zero, the middle term drops out entirely. Simplifying the remaining terms gives:
$$\text{Error} = \sum_{x \in S_k} (x - \mu)^2 + |S_k|(\mu - C)^2$$

The first term is the natural variance of the data (unchangeable). The second term can only be minimized to zero when:
$$C = \mu$$

This algebraically proves that setting the centroid to the cluster mean minimizes the MSE.

### The Accuracy Paradox

I ran this "perfect" math engine across the entire network. Because it optimized specifically for each layer, it achieved the lowest possible MSE in the world. But look at the final model accuracy:

- **INT8 (Standard Grid, 8-bit):** 75.07% accuracy
- **OPTIM (Lloyd-Max, 8-bit):** 74.98% accuracy
- **NF4 (Normal Distribution, 4-bit):** 74.31% accuracy
- **OPTIM (Lloyd-Max, 4-bit):** 74.08% accuracy

Even though OPTIM had a superior MSE, **it lost to standard INT8**. 

*Why?* Because Lloyd-Max places codebook values where the massive clump of weights is, completely ignoring the rare extreme outliers. The neural network physically needs those extreme imperfections to function correctly. MSE is a deceptive metric.

---

## 4. The Star Graph & Kurtosis Crossover (Phase 3)

To summarize the entire research question visually, I plotted the accuracy of every format against its bit-width constraint.

![Accuracy vs. Bits Star Graph](/assets/img/star_graph_accuracy.png)

To isolate *why* this happens, I generated synthetic distributions where I manually controlled the excess kurtosis ($\kappa$):

$$\kappa = \frac{1}{\sigma^4} \left( \frac{1}{N} \sum_{i=1}^{N} (x_i - \mu)^4 \right) - 3$$

Blending Uniform distributions with Cauchy distributions to create sliding fat tails, I mapped the crossover point where standard INT8 gets destroyed by extreme outliers, while dynamic formats like FP8 handle them effortlessly.

![Format Supremacy Inversion as kurtosis increases](/assets/img/kurtosis_crossover.png)

---

## 5. Conclusion & Limitations

I discovered that minimizing MSE is a trap. A mathematically "perfect" codebook fails in the real world because it smooths out the chaotic outliers that the neural network relies on to make predictions. Standard formats like INT8 and NF4 are actually already operating at near-maximum efficiency for their respective distributions.

However, this work comes with strict limitations:
1. **Scale of the Network:** I trained a small CNN with 600,000 parameters on CIFAR-10. This is fundamentally different from a 70-Billion parameter Transformer, which features much more extreme activation outliers.
2. **Post-Training Quantization (PTQ) Only:** I applied codebooks *after* training rather than using Quantization-Aware Training (QAT).
3. **Simulation vs. Execution:** My custom Python encoders faked quantization by casting back to FP32, meaning I proved memory compression rather than raw hardware execution speedup.

The repository containing all custom encoders, math engines, and Typst report files is fully open-source on GitHub!