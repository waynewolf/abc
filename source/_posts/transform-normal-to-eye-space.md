---
title: transform normal to eye space
date: 2013-05-30 10:50:50
tags:
- graphic
---

## Problem
Vertex position coordinates can be transformed from object space to eye space using model-view matrix. However, normals can not be transformed in that way.

The two pictures shows what happened if we transform normal using model-view matrix:
![transform normal](/assets/transform-normal.gif "transform normal using model view matrix")

In eye space, tangent direction is correct, but normal is not perpendicular to tangent. Why is that happening?

## Why
The math equation can explain it easily. Assume T is tangent, MV is model-view matrix. P1, P2 are two vertices used to calculate tangent.

    T = P2 - P1
    T' = T * MV = (P2 - P1) * MV = P2 * MV - P1 * MV = P2' - P1'
　　

T' keeps tangent attributes, while normal does't. You can find point Q1, Q2 to let N=Q2-Q1, after transformation, Q2'-Q1' is not perpendicular to T'. The angle relationship doesn't retain after object space to view space transformation.

## How to transform normal
We need to calculate normal transformation from object to eye space, after which the transformed normal is still perpendicular to transformed tangent.
Assume the normal transformation is G, model view transformation is V, normal is perpendicular to tangent, so

    N'.T' = (GN).(VT) = 0

Vector dot product written as matrix multiplication:

(GN).(VT) = (GN)<sup>T</sup>(VT) =  (N<sup>T</sup>G<sup>T</sup>)(VT) = N<sup>T</sup>G<sup>T</sup>VT = 0

Notice that N<sup>T</sup>T is 0(N is perpendicular to T, so N.T = 0, hence N<sup>T</sup>T ＝ 0), if G<sup>T</sup>V = I, above equation is true, so G=(V<sup>-1</sup>)<sup>T</sup>. G equals to V only when V(model-view matrix) is orthogonal.

## Conclusion
normal matrix is the transpose of inverse model-view matrix, this is a very important conslusion that you should remember, because it is often used when calculating lighting.

