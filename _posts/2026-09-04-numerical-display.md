---
title: Numerical Display
date: 2026-09-04
categories: [Breadboard CPU]
description: Design for 7-segment numerical display
---

## Background and Terminology

The ASAP-3 uses *common cathode* 7-segment displays, which operate by asserting +5V on the segments that you want to light up. For these displays, then, the segmental patterns for each decimal digits are as follows:

### Table 1. Common cathode 7-segment display patterns

![a display digit with labeled segments](/assets/images/7-segment.png){: width="20%"}

| Digit | a | b | c | d | e | f | g | 
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 0 | 1 | 1 | 1 | 1 | 1 | 1 | 0 |
| 1 | 0 | 1 | 1 | 0 | 0 | 0 | 0 |
| 2 | 1 | 1 | 0 | 1 | 1 | 0 | 1 |
| 3 | 1 | 1 | 1 | 1 | 0 | 0 | 1 |
| 4 | 0 | 1 | 1 | 0 | 0 | 1 | 1 |
| 5 | 1 | 0 | 1 | 1 | 0 | 1 | 1 |
| 6 | 1 | 0 | 1 | 1 | 1 | 1 | 1 |
| 7 | 1 | 1 | 1 | 0 | 0 | 0 | 0 |
| 8 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
| 9 | 1 | 1 | 1 | 1 | 0 | 1 | 1 |

