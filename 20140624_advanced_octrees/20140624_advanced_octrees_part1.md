# Introduction
An *Octree* is a recursive, axis-aligned, spatial partitioning data structure commonly used in computer graphics to optimize collision detection, nearest neighbor search, frustum culling and more. Conceptually, an Octree is  a simple data structure. However, when digging deeper into the literature one will find many interesting, not very well-known techniques for optimizing and extending Octrees. This is why I decided to write a series of blog posts about Octree techniques not widely covered on the Internet. This series will consist of five posts covering the following topics:

1. Preliminaries, insertion strategies and maximum tree depth
1. Different node representations for memory footprint reduction
1. Non-static Octrees to support moving objects and inserting objects not falling into the root cell
1. Loose Octrees for optimizing insertion and culling hotspots
1. Accessing a node's neighbors

I'll publish the posts of this series one by one during the next few weeks, starting today with the first one on preliminaries, insertion strategies and an upper bound for the maximum Octree depth. Thanks for reading!

# Preliminaries
An Octree hierarchically subdivides a finite 3D volume into eight disjoint *octants*. In the following, octants are also called *nodes* in the context of tree data structures and *cells* in the context of space. An Octree's root cell encloses the entire world. Sometimes, Octrees are introduced as subdividing space into cubes instead of arbitrarily sized boxes. Generally, arbitrarily sized boxes work equally well and there's no need to adjust the root cell's size to have the shape of a cube. However, cube-shaped cells slightly speed-up cell subdivision computations and the cell size can be stored as just one `float` per node instead of three. The general structure of an Octree is illustrated in the figure below.

![](https://geidav.files.wordpress.com/2014/07/octree.png)

An Octree is constructed by recursively subdividing space into eight cells until the remaining number of objects in each cell is below a pre-defined threshold, or a maximum tree depth is reached. Every cell is subdivided by three axis-aligned planes, which are usually placed in the middle of the parent node. Thus, each node can have up to eight children. The possibility not to allocate certain child nodes allows, in contrast to regular grids, to store sparsely populated worlds in Octrees.

# Insertion strategies
Points are dimensionless and thereby have no spatial extent. Thus, if points are stored in an Octree, they can be unambiguously assigned to exactly one node. However, if objects with spatial extent like polygons are stored in an Octree, their midpoint can fall into a cell while the object it-self straddles the cell boundary. In that case there are basically three options.

1. The object in question is split along the boundaries of the straddling cells and each part is inserted into its corresponding cell. This approach has two disadvantages. First, the splitting operation might imply costly computations. Second, the data structures are more complicated, because the split-off objects need to be stored somewhere.

1. The object in question is added to each cell it straddles. This option is disadvantageous when the Octree needs to be updated, because it can contain duplicate references to the same object. Furthermore, the culling performance gets worse, because the same object might be found in more than one visible cell. Additionally, special care must be taken when subdividing cells. If after subdividing a cell all objects straddle the very same newly created sub-cell(s), all objects will be inserted again into the same sub-cell(s) causing yet another subdivision step. This results in an infinite loop, only stopped when the maximum tree depth is reached.

1. The object in question is stored in the smallest Octree cell it's completely enclosed by. This option results in many unnecessary tests, because objects are stored in inner nodes instead of leaf nodes in case they straddle any child cell further down the tree.

In some of the following posts of this series I'll come back to the different insertion strategies and show in which situation which of the strategies is advantageous. Especially, *Loose Octrees* are a particularly nice way of overcoming most of the downsides discussed above.

# Maximum tree depth
Let's assume an Octree contains $M$ points. As described in the previous section, each of the points can only fall exactly into one node. Is it possible to establish a relation between the number of points $M$ and the maximum Octree depth $D_\text{max}$? It turns out, that the number of Octree nodes (=> the Octree depth) is not limited by the number of points. The reason is, that if the points are distributed closely enough in multiple widespread clusters, the number of Octree nodes can grow arbitrarily large. Look at the following figure for an illustration.

![](https://geidav.files.wordpress.com/2014/07/octree_clusters.png)

In order to split any of the two point clusters, the Octree must be subdivided a few times first. As the points inside the clusters can be arbitrarily close and the clusters can be arbitrarily far away from each other, the number of subdivision steps, and therefore the number of Octree nodes is not limited by the size of the point set. That shows that generally, Octrees cannot be balanced as we are used to from traditional tree data structures.

Nevertheless, we can come up with another upper bound for the maximum tree depth. Let's assume a cubic Octree for simplicity. Given the minimum distance $d_\text{min}$ between any two points in the point set and the side length of the root cell $s$, it can be shown that the maximum Octree depth is limited by $\log\frac{s}{d_\text{min}}+\log\sqrt{3}\geq D_\text{max}$. The following proof for this upper bound is rather simple.
The maximum distance between any two points in a cell at depth $k$ is given by $\sqrt{3(s/2^k)^2}=\sqrt{3}\frac{s}{2^k}$. Any inner node encloses at least two points, otherwise it would be a leaf node. Hence, the maximum distance between any two points in this cell is guaranteed to be bigger than the minimum distance $d_\text{min}$ between any two points of the point set Therefore, it holds that $d_\text{min}\leq\sqrt{3}\frac{s}{2^k}\Leftrightarrow k\leq\log\sqrt{3}+\log\frac{s}{d_\text{min}}$.

That’s it for today. Stay tuned for the next article!