# Splitting a polygon by a line
In this post I present an algorithm to split arbitrary [*simple polygons*](http://en.wikipedia.org/wiki/Simple_polygon) by lines. My implementation is based on ideas from the article *Spatial Partitioning of a Polygon by a Plane* by George Vanecek, published in the book *Graphics Gems V*. The intention behind this post is covering all the details which the original article omitted.

## Introduction
*Splitting* a polygon by a line is very closely related to *clipping* a polygon against a clip region, like a rectangle or another polygon. Splitting a polygon is easier and at the same time harder than clipping a polygon. It's easier because you only clip against a single line and it's harder because the resulting cut-off polygons cannot simply be discarded. It turns out, that splitting *convex* polygons is a lot easier than splitting *concave* polygons. That is because splitting convex polygons results only in at most two new polygons, whereas splitting concave polygons can result in arbitrary many new polygons. Look at the following figure for an illustration.

![Convex vs concave polygon](https://dl.dropboxusercontent.com/u/1174401/convex_vs_concave_poly.png)

Coming up with a correct implementation is tricky, as there're many corner cases that must be considered. For example, whenever the split line doesn't cross a polygon edge directly, but only touches one or even two of its vertices, things start getting tedious. In the following, first, an algorithm for splitting simple, convex polygons is outlined. Afterwards, the algorithm is extended to also work for simple, concave polygons.

## Convex polygons
Splitting convex polygons is pretty simple. All that has to be done is iterating over the polygon's vertices, checking on which side of the split line they lie and adding them accordingly either to the left or the right result polygon. In case an edge crosses the split line, additionally, the intersection point must be added to both result polygons. Special care must be taken when an intersection point is equal to one of the polygon's vertices. In that case, not the intersections point, which is equal to the vertex lying on the split line, but the vertex it-self must be added to both result polygons. When for the second time a polygon edge crosses the split line, we know that the first half of the polygon is already completely split off.

## Concave polygons
In contrast to splitting convex polygons, splitting concave polygons requires some more algorithmic effort. When the split line crosses a concave polygon for the second time, it doesn't necessarily mean that the current polygon is now closed and thus split off. On the contrary, it might be that another new polygon to be split off just opened. To account for that, first, I tried to iterate over the polygon's edges while carrying along a stack of open polygons. Whenever, I came accross a vertex which belonged to the open polygon on the top of the stack I knew it's now closed. This approach worked out well until I tried accounting for all the corner cases. Finally, I decided to use the method of the article mentioned in the introduction.

### Algorithm idea
It turns out that the the intersection points between the split line and the polygon are key to the splitting algorithm.  By pairing up 2-tuples of intersection points along the split line, the polygon edges closing the split off polygons can be determined. Conceptually, every two intersection points along the split line form a pair of edges, each closing one side of the two neighboring polygons. In a perfect world, the intersection points could be easily paired up after sorting them along the split line relative to the split line's starting point (cf. the left figure below). Unfortunately, in the real world special care must be taken when the split line doesn't cross a polygon edge in middle, but only touches the start and/or end vertex. In such cases the pairing up of *source* (red) and *destination* points (green) along the split line gets trickier (cf. the figure in the middle blow). As you can see, only certain intersection points qualify as sources and destinations and some intersection points even qualify as both (yellow, cf. the right figure below). Luckily, it's possible to account for all of these cases as we'll see later.

![Edge classification cases](https://dl.dropboxusercontent.com/u/1174401/edge_classification_cases.png)

### Computing intersection points
We start with computing the intersection points between the polygon and the split line. To ease slicing the polygon later on, it's handy to represent the polygon as a doubly linked list. Then we iterate over each polygon edge `e` and determine on which side of the split line the edge's start vertex `e.start` and end vertex `e.end` lies. If `e.start` lies on the split line, `e` is simply added to the array `EdgesOnLine`, which contains all edges starting on the split line. If `s.start` and `s.end` lie on opposite sides of the split line, a new edge starting at the intersection point is added to the polygon and inserted into `EdgesOnLine`. The edges in this array are of interest because they start at intersection points with the split line. The algorithm is outlined below.

```cpp
for (auto &edges : poly)
{
    LineSide sideStart = GetLineSide(SplitLine, edge.Start);
    LineSide sideEnd = GetLineSide(SplitLine, edge.End);

    if (sideStart == ON)
    {
        EdgesOnLine.Append(edge);
    }
    else if (sideStart != sideEnd && sideEnd != ON)
    {
        Point2 ip = IntersectLines(SplitLine, edge);
        Edge *newEdge = poly.InsertEdge(edge, ip);
        EdgesOnLine.append(newEdge);
    }
}
```

### Sorting intersection points
After all polygon edges have been intersected with the split line, the edges in the `EdgesOnLine` array are sorted by the distance between their start position and the split line's start position. Sorting the the `EdgesOnLine` array is crucial for pairing up source and destination edges by simply iterating over the array as we see in the following section.

```cpp
std::sort(EdgesOnLine.begin(), EdgesOnLine.end(), [&](PolyEdge *e0, PolyEdge *e1)
{
    return (PointDistance(SplitLine.Start, e0->StartPos) <
            PointDistance(SplitLine.Start(), e1->StartPos));
});
```

### Finding source and destination edges
All edges in the `EdgesOnLine` array are either source edges, destination edges or both. They need to be classified so that we're able to determine between which pairs of edges a bridge must be created to split the polygon. The source/destination classification is the most important part of the algorithm and the hardest to get right because of the corner cases.  
Any edge `e` in `EdgesOnLine` has a start position `v` lying on the split line. Each `v` has a predecessor `P(v)` and a successor `S(v)`. `P(v)` and `S(v)` can lie on the *left* hand side (`L`), on the *right* hand side (`R`) or *on* the split line (`O`). Hence, there's a total of nine configuration possibilities that must be evaluated for if they can be source, destination or both. The configurations are depicted in the figure below. The vertex order is shown in the caption of each configuration.

![Edge configurations](https://dl.dropboxusercontent.com/u/1174401/edge_configurations.png)

Careful examination by looking at examples classifies each configuration as source, destination or both. The configurations' winding orders are key for the classification. Imagine to move along the split line from the start to the end position. The winding order indicates if we're currently inside the polygon or not. Let's take examplary the `LOR` and `ROL` case. The `LOR` case is only encoutered when transitioning from the outside to the inside of a polygon. Hence, it's a source configuration. In contrast, the `ROL` case is only encoutered when transitioning from the inside of the polygon to the outside (hence, it's a destination configuration). Below are sample polygons for the classification of each source and each destination configuration depicted.

![Classification cases](https://dl.dropboxusercontent.com/u/1174401/classification_cases.png)

It follows that `LOR`, `ROR`, `LOL`, `OOR` and `LOO` are source configurations and `ROL`, `ROR`, `LOL`, `ROO` and `OOL` are destination configurations. Note, that these two sets are not disjoint: `ROR` and `LOL` are source and destination configurations. However, these two configurations can only be sources if they have served as destinations before!  
Another corner case occurs when two successive edge start positions lie on the split line. Look at the example depicted in the figure below. There're two source edges (red) of which the second one must be used as source, eventhough it comes second in the `EdgesOnLine` array.

![Source/destination selection problems](https://dl.dropboxusercontent.com/u/1174401/src_dst_problem.png)

The `EdgesOnLine` array contains three edges: two source edges (red) and one destination edge (green). When splitting the polygon it's crucial to pair up the the second and the third edge, not the first and the third. So how do we figure out which source edge to select when iterating over the `EdgesOnLine` array? It turns out that the distance between the start positions of the two successive source edges in question and the start position of the split line can be used. An `OOR` and `LOO` source edge is only selected if the following candidate edge isn't further away from the start of the split line and therefore closer to the next destination.

### Rewiring the polygon
After we figured out how the classification of source and destination edges works, we can finally formulate the algorithm to split concave polygons. All we need to do is to iterate over the `EdgesOnLine` array pairing up source and destination edges. Each pair denotes a side of the polygon that needs to be split. At this point it pays off that we use a linked list representation for the polygon. All that needs to be done to split the polygon is rewiring it between the source and the destination edge. Therefore, we iterate over the `EdgesOnLine` array, obtain the next source and the next destination edge and insert two new oppositely facing edges between them. This *bridging* operation is depicted in the following figure.

![Edge configurations](https://dl.dropboxusercontent.com/u/1174401/insert_split_bridge.png)

### Collecting the split off polygons
The final step, after all bridge edges have been inserted, is collecting the resulting polygons. For that purpose I use the `Visited` attribute in the edge data structure which indicates if an edge has been visited already and therefore, assigned to one of the result polygons.

## Wrap up
Splitting concave polygons is much more complicated than splitting convex polygons. By sorting the intersection points along the split line and inserting new bridge edges between source and destination intersection points, the polygon can be split along the line. Representing the polygon as a doubly linked list is thereby key. The source code of my implementation can be downloaded from [github](https://github.com/geidav/concave-poly-splitter).