---
description: How the functional elements of AppLayer interact with each other
---

# BDK implementation

This chapter aims to explain how the BDK and some of its most important components are implemented from a more conceptual point-of-view, giving an initial idea on how everything comes together to deliver a blazing fast blockchain.

Some subchapters paint a more holistic view of the BDK, as most components are pretty straight-forward to understand. Other subchapters focus on some components that are particularly dense and/or complex enough that they warrant their own separated explanations.

Developers are expected to read the [Doxygen](https://doxygen.nl) documentation alongside this one to further understand how the project works from a more technical point-of-view - as it is constantly being iterated upon, we make extensive use of comments and self-documenting code to keep up with the internals.

We will focus on the original implementation of the BDK (the C++ one, commonly referred to as "bdk-cpp"). Future implementations, if/when they appear, may be handled separately.