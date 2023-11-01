# 3c Sampling Experiments on Diffusion Curvature

::: {.cell 0=‘d’ 1=‘e’ 2=‘f’ 3=‘a’ 4=‘u’ 5=‘l’ 6=‘t’ 7=‘*’ 8=’e’ 9=’x’
10=’p’ 11=’ ’ 12=’3’ 13=’c’ 14=’*’ 15=‘e’ 16=‘x’ 17=‘p’ 18=‘e’ 19=‘r’
20=‘i’ 21=‘m’ 22=‘e’ 23=‘n’ 24=‘t’ 25=‘s’ 26=’ ’ 27=‘h’ 28=‘i’ 29=‘d’
30=‘e’ execution_count=2}

``` python
## Standard libraries
import os
import math
import numpy as np
import time
# Configure environment
os.environ['XLA_PYTHON_CLIENT_PREALLOCATE']='false' # Tells Jax not to hog all of the memory to this process.

## Imports for plotting
import matplotlib.pyplot as plt
%matplotlib inline
from matplotlib.colors import to_rgba
import seaborn as sns
sns.set()

## Progress bar
from tqdm.auto import tqdm, trange

## project specifics
import diffusion_curvature
from diffusion_curvature.datasets import *
from diffusion_curvature.graphs import *
from diffusion_curvature.core import *
import jax
import jax.numpy as jnp
jax.devices()

%load_ext autoreload
%autoreload 2
```

:::

## Dimensional Analysis of Planes

How does it perform with planes of varying dimensions?

Using clustering within the manifold:

``` python
ds = [3,4,5,6]
planes = [plane(1000*2**(d-2), d) for d in ds]
for i, d in enumerate(ds):
    G = get_alpha_decay_graph(planes[i], decay=None, knn=15, anisotropy=1, )
    DC = DiffusionCurvature(laziness_method="Entropic",points_per_cluster=500,comparison_space_size_factor=1,comparison_method="Subtraction", flattening_method="Fixed")
    ks = DC.curvature(G, t=8, dim=d, knn=15)
    print("dimension",d,": Curvature of Plane is ",ks[0])
```

      0%|          | 0/4 [00:00<?, ?it/s]

    dimension 3 : Curvature of Plane is  0.22488499
    dimension 4 : Curvature of Plane is  -0.0021333694
    dimension 5 : Curvature of Plane is  -0.1174202
    dimension 6 : Curvature of Plane is  -0.098750114

      0%|          | 0/8 [00:00<?, ?it/s]

      0%|          | 0/16 [00:00<?, ?it/s]

      0%|          | 0/32 [00:00<?, ?it/s]

Without clustering

``` python
ds = [3,4,5,6]
planes = [plane(1000*2**(d-2), d) for d in ds]
for i, d in enumerate(ds):
    G = get_alpha_decay_graph(planes[i], decay=None, knn=15, anisotropy=1, )
    DC = DiffusionCurvature(laziness_method="Entropic",points_per_cluster=None,comparison_space_size_factor=1,comparison_method="Subtraction", flattening_method="Fixed")
    ks = DC.curvature(G, t=8, dim=d, knn=15)
    print("dimension",d,": Curvature of Plane is ",ks[0])
```

      0%|          | 0/1 [00:00<?, ?it/s]

    dimension 3 : Curvature of Plane is  -0.113577366
    dimension 4 : Curvature of Plane is  -0.031496525
    dimension 5 : Curvature of Plane is  -0.05647278
    dimension 6 : Curvature of Plane is  0.07413292

      0%|          | 0/1 [00:00<?, ?it/s]

      0%|          | 0/1 [00:00<?, ?it/s]

      0%|          | 0/1 [00:00<?, ?it/s]

**Conclusion**: when using more points, there’s less variance between
dimensions — though still a slightly alarming amount of variance within
them. There doesn’t appear to be any pervasive bias induced by
dimensionality in either setting. That said, this is the *best possible*
condition, as it is literally comparing a plane to a plane.

## Planes under different sampling

``` python
sampled_plane_ks = []
t = 8
for i in trange(1000):
    X_plane = plane(1000,2)
    G = get_alpha_decay_graph(X_plane, decay=None, knn=15, anisotropy=1, )
    DC = DiffusionCurvature(laziness_method="Entropic",points_per_cluster=None,comparison_space_size_factor=1,comparison_method="Subtraction", flattening_method="Fixed")
    ks = DC.curvature(G, t=t, dim=2, knn=15)
    sampled_plane_ks.append(ks[0])
```

      0%|          | 0/1000 [00:00<?, ?it/s]

``` python
# plot a histogram of the curvature values
plt.hist(sampled_plane_ks, bins=100)
plt.title(f"Ks of Plane, with No Clustering, Subtraction, t={t}")
```

    Text(0.5, 1.0, 'Ks of Plane, with No Clustering, Subtraction, t=8')

![](3c%20Sampling%20Experiments_files/figure-markdown_strict/cell-6-output-2.png)

That’s not looking very good. The randomness of the plane sampling,
combined with the randomness of the comparison space has created a lot
of variability.

I see two strategies to address this: using a higher *t*, and comparing
to a grid.

``` python
sampled_plane_ks = []
t = 8
for i in trange(1000):
    X_plane = plane(1000,2)
    G = get_alpha_decay_graph(X_plane, decay=None, knn=15, anisotropy=1, )
    DC = DiffusionCurvature(laziness_method="Entropic",points_per_cluster=None,comparison_space_size_factor=1,comparison_method="Subtraction", flattening_method="Fixed", use_grid=True)
    ks = DC.curvature(G, t=t, dim=2, knn=15)
    sampled_plane_ks.append(ks[0])
```

      0%|          | 0/1000 [00:00<?, ?it/s]

``` python
# plot a histogram of the curvature values
plt.hist(sampled_plane_ks, bins=100)
plt.title(f"Ks of Plane, with Grid, No Clustering, Subtraction, t={t}")
```

    Text(0.5, 1.0, 'Ks of Plane, with Grid, No Clustering, Subtraction, t=8')

![](3c%20Sampling%20Experiments_files/figure-markdown_strict/cell-8-output-2.png)

Now this is odd. It appears the grid *biases* the results; it must have
a higher-than-normal entropy, making everything appear more positive
while it, the comparison, looks falsely negative. *What’s up with this?*

On a more positive note, using a grid did shave off 0.2 variance.

Hypothesis 1: It’s the kernel we’re using. That darn alpha-decay kernel
is somehow changing the shape of the grid. Disabling the decay should
remedy the problem. To be doubly sure that’s working, I can construct a
kernel with my code.

1.  Decay=None does nothing – that’s what we were using before.
2.  But perhaps it has to do with the knn value! In a grid, the 15th
    nearest neighbor may be further than it is on a uniformly sampled
    surface, because the points are arranged in squares rather than
    circles.

``` python
sampled_plane_ks = []
t = 8
for i in trange(1000):
    X_plane = plane(1000,2)
    G = get_alpha_decay_graph(X_plane, decay=None, knn=10, anisotropy=1, )
    DC = DiffusionCurvature(laziness_method="Entropic",points_per_cluster=None,comparison_space_size_factor=1,comparison_method="Subtraction", flattening_method="Fixed", use_grid=True)
    ks = DC.curvature(G, t=t, dim=2, knn=10)
    sampled_plane_ks.append(ks[0])
```

      0%|          | 0/1000 [00:00<?, ?it/s]

``` python
# plot a histogram of the curvature values
plt.hist(sampled_plane_ks, bins=100)
plt.title(f"Ks of Plane, with Grid, No Clustering, Subtraction, t={t}, knn=10")
```

    Text(0.5, 1.0, 'Ks of Plane, with Grid, No Clustering, Subtraction, t=8, knn=10')

![](3c%20Sampling%20Experiments_files/figure-markdown_strict/cell-10-output-2.png)

Support for the knn hypothesis. Changing k from 15 to 10 increased the
perceived negativity of the grid’s curvature.

This would likely be best avoided by *not* using a knn grid; or using
some average of distances, rather than the concrete *distance from the
kth* nearest neighbor.

It would also, if the hypothesis is shrewd, diminish in effect the
higher the k.

``` python
sampled_plane_ks = []
t = 8
for i in trange(1000):
    X_plane = plane(1000,2)
    G = get_alpha_decay_graph(X_plane, decay=None, knn=30, anisotropy=1, )
    DC = DiffusionCurvature(laziness_method="Entropic",points_per_cluster=None,comparison_space_size_factor=1,comparison_method="Subtraction", flattening_method="Fixed", use_grid=True)
    ks = DC.curvature(G, t=t, dim=2, knn=30)
    sampled_plane_ks.append(ks[0])
```

      0%|          | 0/1000 [00:00<?, ?it/s]

``` python
# plot a histogram of the curvature values
plt.hist(sampled_plane_ks, bins=100)
plt.title(f"Ks of Plane, with Grid, No Clustering, Subtraction, t={t}, knn=30")
```

    Text(0.5, 1.0, 'Ks of Plane, with Grid, No Clustering, Subtraction, t=8, knn=30')

![](3c%20Sampling%20Experiments_files/figure-markdown_strict/cell-12-output-2.png)

Indeed, using a larger knn value decreased the descrepency considerably.
*But it’s still there!*

As an ablation, here’s this same experiment on a 5000 point plane.

Here’s the k=15 version: (With GPU, it jumps to about 10 minutes to run
1000 trials. Not bad. Thank ya, Nvidia!)

``` python
sampled_plane_ks = []
t = 8
for i in trange(1000):
    X_plane = plane(5000,2)
    G = get_alpha_decay_graph(X_plane, decay=None, knn=15, anisotropy=1, )
    DC = DiffusionCurvature(laziness_method="Entropic",points_per_cluster=None,comparison_space_size_factor=1,comparison_method="Subtraction", flattening_method="Fixed", use_grid=True)
    ks = DC.curvature(G, t=t, dim=2, knn=15)
    sampled_plane_ks.append(ks[0])
```

      0%|          | 0/1000 [00:00<?, ?it/s]

``` python
# plot a histogram of the curvature values
plt.hist(sampled_plane_ks, bins=100)
plt.title(f"Ks of Plane, with Grid, No Clustering, Subtraction, t={t}, knn=15")
```

    Text(0.5, 1.0, 'Ks of Plane, with Grid, No Clustering, Subtraction, t=8')

![](3c%20Sampling%20Experiments_files/figure-markdown_strict/cell-14-output-2.png)

``` python
sampled_plane_ks = []
t = 8
for i in trange(1000):
    X_plane = plane(5000,2)
    G = get_alpha_decay_graph(X_plane, decay=None, knn=30, anisotropy=1, )
    DC = DiffusionCurvature(laziness_method="Entropic",points_per_cluster=None,comparison_space_size_factor=1,comparison_method="Subtraction", flattening_method="Fixed", use_grid=True)
    ks = DC.curvature(G, t=t, dim=2, knn=30)
    sampled_plane_ks.append(ks[0])
```

      0%|          | 0/1000 [00:00<?, ?it/s]

``` python
# plot a histogram of the curvature values
plt.hist(sampled_plane_ks, bins=100)
plt.title(f"Ks of Plane, with Grid, No Clustering, Subtraction, t={t}, knn=30")
```

    Text(0.5, 1.0, 'Ks of Plane, with Grid, No Clustering, Subtraction, t=8, knn=30')

![](3c%20Sampling%20Experiments_files/figure-markdown_strict/cell-16-output-2.png)

The grid, if it can be made to work, greatly reduces the variance in
reported curvatures. To work with it, I see two immediate options:

1.  Adopting a non-knn kernel – something more sophisticated – that
    weighs across the distances of all of the k nearest points, and not
    merely the kth point. If used simultaneously on real data and the
    comparison space, this would allow us to use the grid.
2.  Instead of changing our kernel to match the hyper-uniform sampling
    of the grid, whose chief advantage is predictability, we could
    *average* the results of a large number of uniform samplings. We can
    take the average entropy over N uniform samplings for a grid of
    likely pairings between k, t, and d.

I favor the latter approach, as it would also reduce the runtime of the
algorithm, by precomputing the expected uniform samplings. If parameters
are chosen outside of the precomputed grid, you can revert to a single
sampling, as we presently do it. Additionally, this would allow fast
matching of comparison spaces with graphs. At low *t* values, the
diffusion entropy should be approximately equal to the *flat* entropy,
modulo the kernel bandwidth, thus allowing us to estimate that *k**n**n*
parameter.

The feasibility question is this: how many pairings do we need?

``` python
num_ks = 30
num_ts = 50
num_ds = 10 # anything much higher dimensional is impossible to get enough samples from
num_trials = num_ks * num_ts * num_ds
num_trials
```

    15000

Each of those could be done in at most 5 minutes, costing

``` python
str(num_trials/12/24)[:4] + " days"
```

    '52.0 days'

Of course, I can parallelize that across Yale’s clusters, cutting it
down to just a couple of days. That seems feasible.

The other feasibility check is whether the number of points in the
comparison space changes the entropy. It shouldn’t. Let’s check:

``` python
ks_bigs = []
ks_smalls = []
for i in range(100):
    big_plane = plane(5000,2)
    small_plane = plane(500,2)
    DC = DiffusionCurvature(laziness_method="Entropic",points_per_cluster=None,comparison_space_size_factor=1,comparison_method="Subtraction", flattening_method="Fixed", use_grid=False)
    G_big_plane = get_alpha_decay_graph(big_plane, knn=15, anisotropy=1, decay=None)
    G_small_plane = get_alpha_decay_graph(small_plane, knn=15, anisotropy=1, decay=None)
    ks_big = DC.unsigned_curvature(G_big_plane, t=8, idx=0)
    ks_small = DC.unsigned_curvature(G_small_plane, t=8, idx=0)
    ks_bigs.append(ks_big) 
    ks_smalls.append(ks_small)
```

``` python
# show the mean and standard deviation of ks_bigs and ks_smalls
print("mean of ks_bigs:", np.mean(ks_bigs))
print("mean of ks_smalls:", np.mean(ks_smalls))
print("std of ks_bigs:", np.std(ks_bigs))
print("std of ks_smalls:", np.std(ks_smalls))
```

    mean of ks_bigs: 5.059733
    mean of ks_smalls: 5.0795994
    std of ks_bigs: 0.07575969
    std of ks_smalls: 0.0928274

So there is a difference arising from the number of points used. Likely
this is because the diffusion, though concentrated in the center of our
plane, has lots of ‘close to zero’ values that have spread across the
manifold.

I tried changing the entropy calculation to zero out elements below an
epsilon threshold (set to 10e-5); this helps some, but for large
discrepancies in the number of points there’s still a .02 discrepancy.
Perhaps that’s close enough it can be tolerated – it’s certainly less
than the variance within different uniform samplings of the plane.

The next step is to modify the ‘Fixed’ comparison space construction to
first load a database of flat entropies (on initialization of the
class?), check if the current parameters are within the database, and,
if not, average the uniform sampling N times.

## Effects of *t*

``` python
sampled_plane_ks = []
t = 25
for i in trange(1000):
    X_plane = plane(1000,2)
    G = get_alpha_decay_graph(X_plane, decay=None, knn=15, anisotropy=1, )
    DC = DiffusionCurvature(laziness_method="Entropic",points_per_cluster=None,comparison_space_size_factor=1,comparison_method="Subtraction", flattening_method="Fixed", use_grid=True)
    ks = DC.curvature(G, t=t, dim=2, knn=30)
    sampled_plane_ks.append(ks[0])
```

      0%|          | 0/1000 [00:00<?, ?it/s]

``` python
# plot a histogram of the curvature values
plt.hist(sampled_plane_ks, bins=100)
plt.title(f"Ks of Plane, with Grid, No Clustering, Subtraction, t={t}, knn=15")
```

    Text(0.5, 1.0, 'Ks of Plane, with Grid, No Clustering, Subtraction, t=25, knn=15')

![](3c%20Sampling%20Experiments_files/figure-markdown_strict/cell-22-output-2.png)

``` python
np.std(sampled_plane_ks)
```

    0.054011818

# Samplings of Curved Surfaces

The plane can only tell us so much, since what’s all-important is *how
much these variances differ* with respect to other curvature values.
