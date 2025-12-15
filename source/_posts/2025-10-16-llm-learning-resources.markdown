---
layout: post
title: "LLM Learning Resources"
date: 2025-10-16 21:54:04 -0700
comments: true
categories:
---

This post lists resources that I find useful during my journey of learning LLM as a system enginer.

<!-- more -->

## Neural Networks

LLMs are large neural networks so having a basic understanding of what neural networks are is helpful.

#### Victor Zhou's Neural Networks From Scratch

[Neural Networks From Scratch](https://victorzhou.com/series/neural-networks-from-scratch/) is a 4-post series
that introduces classic neural networks, recurrent neural networks (RNNs) and convolutional neural networks (CNNs).
It doesn't require any prior knowledge except for some math. One good thing about this series is that it's very hands-on and you will learn, step-by-step, how to write a simple nueral network from scratch using only numpy to solve a real problem.

#### Michael Nielsen's Visual proof that neural nets can compute any function

[Visual proof that neural nets can compute any function](http://neuralnetworksanddeeplearning.com/chap4.html) gives a visual explanation of the [universal approximation theorem](https://en.wikipedia.org/wiki/Universal_approximation_theorem) that neural networks can be used to approximate any continuous function to any desired precision.

## LLM

#### Andrej Karpathy's Neural Networks: Zero to Hero

[Neural Networks: Zero to Hero](https://karpathy.ai/zero-to-hero.html) is a hands-on course that starts with building a simple neural network from scratch and ends with building a GPT from scratch. It's a great course to learn LLM progressively.

### Inference

The best way to learn LLM inference is writing an inference engine from scratch IMO and this helps you better understand model architectures as well.

#### Andrej Karpathy's llama2.c

[llama2.c](https://github.com/karpathy/llama2.c) is an inference engine for Llama 2 in one file of pure C. It can give you a taste of what LLM inference looks like. Based on it, I wrote a [Rust implementation](https://github.com/jjyao/llm.edu/tree/main/llama2.rs) with optimizations like tensor parallelism.
