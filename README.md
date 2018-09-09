**University of Pennsylvania, CIS 565: GPU Programming and Architecture,
Project 1 - Flocking**

* Zach Corse
  * LinkedIn: https://www.linkedin.com/in/wzcorse/
  * Personal Website: https://wzcorse.com
  * Twitter: @ZachCorse
* Tested on: Windows 10, i7-6700HQ @ 2.60GHz 32GB, NVIDIA GeForce GTX 970M (personal computer)

## README

![gif](images/flocking.gif)

Introduction
------------

In this project I implement a standard flocking algorithm but do so on the GPU. Parallelization of flocking is possible because boid trajectory changes can be computed independently using shared position and velocity memory buffers.

A naive approach to flocking would be to compare each fish or bird (aka boid) to the other N-1 boids in the simulation. However, only the boid's nearest neighbors influence its trajectory. Therefore, this simulation includes a uniform grid optimization, in which boids are binned into cells. Each boid is then only compared to boids inside its nearest neighboring cells. I implement two nearest neighbor searches. The first assumes that each cell is twice the radius of the boid's largest flocking distance measure. This means that, in general, eight cells are scanned for neighbors in each kernel. The second assumes that each cell is equal to the radius of the boid's largest flocking distance measure. This means that, in general, 27 smaller cells are scanned for neighbors in each kernel.

I add an additional optimization to the uniform grid described above. After boids are binned into grid cells, this simulation sorts these grid cells by index and boid label simultaneously, such that by querying the index of this sorted array in a kernel, one can access a grid cell and simultaneosly ask which boid is in that grid cell. From a boid's index, one can query a separate array maintaining boid positions and velocities. The additional optimization instead sorts boid positions and velocities directly as it sorts grid cell indices, thereby eliminating the need to access an intermediary boid label array. Consider this a coherent search on top of a uniform grid.


Performance as a Function of Boid Count
------------

The most basic question one may ask about this simulation's performance is how frames per second (FPS) scales with boid (N) count. Presumably, the more boids one adds to a simulation, the lower the FPS you would expect. This is true of the naive (No Grid) approach, but the coherent and uniform grid cases demonstrate slightly different behavior than expected.

![graph1](images/PerformanceFPSasaFunctionofBoidCountVIS128TBB.png)

As the above graph indicates, there is a dip in performance when N=5000, meaning that performance actually increases with boid count immediately afterwards. This may be a consequence of how memory is allocated on the GPU, but this distribution doesn't appear to vary with GPU blocksize. Additionally, we see a marginal boost in performance using a coherent grid over a uniform grid.

We might expect this performance boost based on the following argument: The uniform grid approach requires two reads from global memory--one to read from a boid index array, and other to access each boid's position and velocity. Additionally, position and velocity memory reads will not be contiguous in this case, since we might be accessing any boid's data (i.e., boids won't necessarily be grouped by index). On the other hand, the coherent approach requires one read from globaly memory (position and velocity) and an additional copy kernel. The read will be over contiguous memory however. One less read from global memory that is over contiguous memory should account for the observed performance boost.

Grid Type Comparison with and without Visualization
------------

As expected, turning off the simulation visualization dramatically increases sim FPS. Coherent and Uniform grid algorithms see a nearly 2X boost in performance. The naive approach only marginally improves in performance.

![graph2](images/GridFPSComparison128TBB.png)

Comparing Neighborhood Search Algorithms
------------

As noted above, I implemented two different neighbor search algorithms. The first will sample the nearest 8 cells (where cell width is twice the radius of boid influence) and the second samples the nearest 27 cells (where cell width is equal to the radius of boid influence). My sampling technique for the former method uses a fast approach in which cells are overindexed by a factor of two. These indices in each dimension, modulo 2, then indicate the quadrant within a cell in which a boid resides, permitting the easy computation of its nearest 8 neighbors. 

It would difficult to guess which of these two neighbor searches would yield a higher FPS. While 8 < 27, the cells are larger and may therefore contain more boids. The graph below indicates that we see a slight performance boost using larger cells but fewer of them.

![graph3](images/PerformanceFPSasaFunctionofCellWidthVIS5KBoids128TBB.png)

Evaluating the Effects of Blocksize (TBB)
------------

Varying memory blocksize (threads per block) does not appear to significantly alter performance (for N=5000). Any variation seen below is within the normal variation observed between runs.

![graph4](images/FPSvsTBBVIS5000Boids.png)

Results & Flocking Behavior
------------

As shown below, boids appear to flock like real fish, birds, and bats! 

![screenshot](images/screenshot.PNG)
