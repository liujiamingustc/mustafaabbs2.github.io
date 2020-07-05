---
layout: posts
title: Analyzing SIMPLEC — the “consistent” simpleFOAM control
---

The previous articles in this series took on debugging the simpleFoam solver through a VSCode debugger. SIMPLEC, a “consistent” implementation of the SIMPLE algorithm can be toggled on and off via the “consistent” switch in your fvSolutions file.

![system/fvSolution](https://cdn-images-1.medium.com/max/2000/1*U_rByI0s4seJJbA9Ja7Jsg.png)*system/fvSolution*

The theory on SIMPLEC is a bit dense — so let’s go through it here.

When we derived our starting decomposed equation for SIMPLE:

![](https://cdn-images-1.medium.com/max/2000/1*KSeNYZ5kgGW7ioLy0wAFrQ.png)

Just as a reminder — A has the diagonal coefficients of velocity U, while H has the non-diagonal coefficients.

The SIMPLE algorithm works on velocity and pressure corrections in an iterative method. Quantities like U*, p* are the predicted value in the first iteration of a loop, and U`, p` are the correction quantities that are used to correct the predicted values.

![](https://cdn-images-1.medium.com/max/2000/1*OUqCGhkTpel7pPP_4uGlFw.png)

A key assumption in the SIMPLE algorithm is that H(U`) = 0 — which means that all the non diagonal coefficients of the correction velocity are 0. This correction velocity is obtained from the Poisson equation for pressure.

This means that the contributions to the correction velocity from all neighbouring cells are dropped.

For **SIMPLE**, the correction velocity equation is derived as:

![](https://cdn-images-1.medium.com/max/2000/1*x8pK1OXExDe9-mhzf5ORSg.png)

The pressure correction term can be large and needs to be under-relaxed with each step for this reason. So the algorithm then ends up doing a step like:

![](https://cdn-images-1.medium.com/max/2000/1*Z4mkRRhyCD_dYPtDk_vUSA.png)

where α is the under-relaxation factor. This means that the pressure correction step is happening much slower than technically possible. How do we get rid of this? By using a consistent formulation: **SIMPLEC**

SIMPLEC has the same basic formulation for correction velocity:

AU` = H1(U`)-∇ p`

Here, we are **not** going to ignore the neighbour velocities (the off-diagonal terms). Let’s rename H(U`) as H1(U`) to avoid confusion with H(U*).

![](https://cdn-images-1.medium.com/max/2000/1*X7D_82eoEtpktqMkgygwEw.png)

Where At = A — H1. Deriving further

![](https://cdn-images-1.medium.com/max/2000/1*qxkszzkOCQ1A5lh7izeZpQ.png)

Compare this to the SIMPLE momentum corrector step to find U:

![](https://cdn-images-1.medium.com/max/2000/1*_rt4hXktIbewk4jebtxblw.png)

You can see that the velocity is corrected by an equivalent of:

![](https://cdn-images-1.medium.com/max/2000/1*_xYOcHyf18tfd_1X6Sr7jg.png)

How does this look like in OpenFOAM?

![SIMPLEC](https://cdn-images-1.medium.com/max/2000/1*16jQ5RMrVrAw2VqmldZG6Q.png)*SIMPLEC*

This is represented by the term (rAU-rAtU())*fvc::grad(p). That’s the correction term for SIMPLEC! Observe the code to see if this matches with theory.

The velocity is then calculated by using the corrected HbyA. It’s that simple! There are no extra equations to be solved (like other SIMPLE variants like SIMPLER), and this may give you gains of 20–30% due to no under-relaxation being required. If you use this article to do a comparison between convergence in SIMPLE and SIMPLEC, do let me know how they fare against each other.
