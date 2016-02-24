In this post I present an algorithm for splitting an arbitrary, [*simple polygon*](http://en.wikipedia.org/wiki/Simple_polygon) by a line. My implementation is based on ideas from the article *Spatial Partitioning of a Polygon by a Plane* by George Vanecek, published in the book *Graphics Gems V*. The intention behind this post is covering all the details which the original article omitted. All polygons are assumed to be oriented counter clockwise. You can find the full source code of my implementation in this [github](https://github.com/geidav/concave-poly-splitter) repository.

# Introduction
*Splitting* a polygon by a line is very closely related to *clipping* a polygon against a clip region, like a rectangle or another polygon. Splitting a polygon is easier and at the same time harder than clipping a polygon. It's easier because you only clip against a single line and it's harder because the resulting polygons cannot simply be discarded. It turns out, that splitting *convex* polygons is a lot easier than splitting *concave* polygons. The reason is that splitting convex polygons results in at most two new polygons, whereas splitting concave polygons can result in arbitrary many new polygons. Look at the following figure for an illustration.

![Convex vs concave polygon](https://geidav.files.wordpress.com/2016/01/convex_vs_concave_poly.png)

Coming up with a correct implementation for concave polygons is tricky, because there are many corner cases that must be considered. In the following, first, an algorithm for splitting  convex polygons is outlined. Afterwards, the algorithm is extended to concave polygons.

# Convex polygons
Splitting convex polygons is pretty simple. Essentially, one iterates over the polygon's vertices, checks on which side of the split line they lie and adds them accordingly either to the left or the right result polygon. When an edge crosses the split line, additionally, the intersection point must be added to both result polygons. Special care must be taken when multiple successive vertices lie on the split line. In that case, due to the convexity property, the entire polygon lies either on the left-hand or on the right-hand side of the polygon. When for the second time a polygon edge crosses the split line, we know that the first half of the polygon is already completely split off.

# Concave polygons
In contrast to splitting convex polygons it requires some more algorithmic effort to split concave polygons. When the split line crosses a concave polygon for the second time, it doesn't necessarily mean that the first half of the polygon is now completely split off. On the contrary, it might be that another new polygon to be split off just *opened*. To account for that, first, I tried to iterate over the polygon edges while carrying along a stack of open polygons. Whenever, I came across a vertex which belonged to the open polygon on the top of the stack I knew it just closed. This approach worked out well until I tried accounting for all the corner cases. Finally, I decided to use the method from the article mentioned in the introduction.

## Idea of the algorithm
The intersection points between the split line and the polygon are key to the splitting algorithm. By pairing up 2-tuples of intersection points along the split line, the edges closing the result polygons can be determined. Conceptually, every two intersection points along the split line form a pair of edges, each closing one side of two neighboring result polygons. In a perfect world, the intersection points could be easily paired up after sorting them along the split line, relative to the split line's starting point (cf. the left figure below). Unfortunately, in the real world special care must be taken when the split line doesn't cross a polygon edge in middle, but only touches the start and/or end vertex. In such cases the pairing up of *source* points (red) and *destination* points (green) along the split line gets trickier (cf. the figure in the middle blow). As you can see, only certain intersection points qualify as sources and destinations and some intersection points even qualify as both (yellow, cf. the right figure below). Luckily, it's possible to account for all of these cases as we'll see later.

![Edge classification cases](https://geidav.files.wordpress.com/2016/01/edge_classification_cases.png)

## Computing the intersection points
To ease the splitting process it's helpful to represent the polygon as a doubly linked list. We compute the intersection points between the split line and the polygon by iterating over each polygon edge `e` and determining on which side of the split line the edge's start vertex `e.Start` and end vertex `e.End` lies. If `e.Start` lies on the split line `e` is added to the array `EdgesOnLine`, which contains all edges starting on the split line. If `e.Start` and `e.End` lie on opposite sides of the split line, a new edge starting at the intersection point is inserted into the polygon and added to `EdgesOnLine`. The edges in this array are of interest, because they start at the intersection points with the split line. The algorithm is outlined below.

```cpp
for (auto &e : poly)
{
    const LineSide sideStart = GetLineSide(SplitLine, e.Start);
    const LineSide sideEnd = GetLineSide(SplitLine, e.End);

    if (sideStart == LineSide::On)
    {
        EdgesOnLine.push_back(e);
    }
    else if (sideStart != sideEnd && sideEnd != LineSide::On)
    {
        Point2 ip = IntersectLines(SplitLine, e);
        Edge *newEdge = poly.InsertEdge(e, ip);
        EdgesOnLine.push_back(newEdge);
    }
}
```

## Sorting the intersection points
After all polygon edges have been intersected with the split line, the edges in the `EdgesOnLine` array are sorted along the split line by the distance between their starting points and the starting point of the split line. Sorting the `EdgesOnLine` array is crucial for pairing up the source and the destination edges by simply iterating over the array as we see in the following section.

```cpp
std::sort(EdgesOnLine.begin(), EdgesOnLine.end(), [&](PolyEdge *e0, PolyEdge *e1)
{
    return (CalcSignedDistance(line, e0->StartPos) <
            CalcSignedDistance(line, e1->StartPos));
});
```

As Glenn Burkhardt pointed out in a comment, the Euclidean distance cannot be used as distance metric because it's unsigned. The Euclidean distance can only be used as long as the split line’s starting point lies outside the polygon. In that case all points are on the same side of the split line’s starting point and the Euclidean distance provides a strict order along the split line. However, if the split line’s starting point is inside the polygon, points on two different sides of the split line’s starting point will both have positive distances, which can result in wrong sort orders. A formula for computing the signed Euclidean distance between two points can be derived from the formula for [scalar projections](https://en.wikipedia.org/wiki/Scalar_projection). In case of co-linear lines (angle between the two vectors is `|1|`) the result equals the signed distance along the split line.

```cpp
static double CalcSignedDistance(const QLineF &line, const QPointF &p)
{
    return (p.x()-line.p1().x())*(line.p2().x()-line.p1().x())+
           (p.y()-line.p1().y())*(line.p2().y()-line.p1().y());
}
```

## Pairing up edges
As we've seen before, all edges in the `EdgesOnLine` array are either source edges, destination edges or both. The edges must be classified accordingly, in order to be able to determine between which pairs of edges a bridge must be created to split the polygon. The source/destination classification is the most important part of the algorithm and the hardest to get right due to plenty of corner cases.  

The starting point `e.Start` of any edge `e` in the `EdgesOnLine` array lies on the split line. Each edge `e` has a predecessor edge `e.Prev` and a successor edge `e.Next`. They can lie either on the *left*-hand side (`L`), on the *right*-hand side (`R`) or *on* the split line (`O`). Hence, there's a total of nine configuration possibilities that must be evaluated in order to find out if an edge is a source, a destination or both. All nine configurations are depicted in the figure below. The vertex order is shown in the caption of each configuration.

![Edge configurations](https://geidav.files.wordpress.com/2016/01/edge_configurations.png)

I simply examined examples to classify each configuration as source, destination or both. The configurations' winding orders are key for the classification. Imagine to move along the split line from its starting point to its endpoint. The winding order at the current intersection point indicates if we're currently inside the polygon or not. Let's take exemplary the `LOR` and `ROL` cases. The `LOR` case is only encountered when transitioning from the outside to the inside of a polygon. Hence, it's a source configuration. In contrast, the `ROL` case is only encountered when transitioning from the inside of the polygon to the outside. Hence, it's a destination configuration. Below are sample polygons depicted for the classification of each source and each destination configuration.

![Classification cases](https://geidav.files.wordpress.com/2016/01/classification_cases.png)

It follows that `LOR`, `ROR`, `LOL`, `OOR` and `LOO` are source configurations and `ROL`, `ROR`, `LOL`, `ROO` and `OOL` are destination configurations. Note, that these two sets are not disjoint: `ROR` and `LOL` are source and destination configurations. However, these two configurations can only be sources if they have served as destinations before!  
Another corner case occurs when two successive edge starting points lie on the split line. Look at the example depicted in the figure below. There are two source edges (red) of which the second one must be used as source, even though it comes second in the `EdgesOnLine` array.

![Source/destination selection problems](https://geidav.files.wordpress.com/2016/01/src_dst_problem.png)

The `EdgesOnLine` array contains three edges: two source edges (red) and one destination edge (green). When splitting the polygon it's crucial to pair up the the second and the third edge, not the first and the third. So how do we figure out which source edge to select when iterating over the `EdgesOnLine` array? It turns out that the distance between the starting points of the two successive source edges in question and the starting point of the split line can be used. An `OOR` and `LOO` source edge is only selected if the following candidate edge isn't further away from the start of the split line and therefore closer to the next destination.

## Rewiring the polygon
After we've figured out how the classification of source and destination edges works, we can finally formulate the algorithm to split concave polygons. All we need to do is to iterate over the `EdgesOnLine` array while pairing up source and destination edges. Each pair denotes one side of the polygon that has to be be split off. At this point it pays off that we use a linked list representation for the polygon. The current polygon side is split off by inserting two new edges between the source and the destination edge. Therefore, we iterate over the `EdgesOnLine` array, obtain the next source and the next destination edge and insert two new, oppositely oriented edges between them. This *bridging* operation is depicted in the following figure.

![Edge configurations](https://geidav.files.wordpress.com/2016/01/insert_split_bridge.png)

The main loop of the rewiring process is depicted below. For reasons of brevity only the overall structure is shown. For more details look into the full implementation in the [github](https://github.com/geidav/concave-poly-splitter) repository.

```cpp
for (size_t i=0; i<EdgesOnLine.size(); i++)
{
    // find source edge
    PolyEdge *srcEdge = useSrc;

    for (; !srcEdge && i<EdgesOnLine.size(); i++)
    {
        // test conditions if current edge is a source
    }

    // find destination edge
    PolyEdge *dstEdge = nullptr;

    for (; !dstEdge && i<EdgesOnLine.size(); )
    {
        // test conditions if current edge is a destination
    }

    // bridge source and destination edges
    if (srcEdge && dstEdge)
        CreateBridge(srcEdge, dstEdge);
}
```

## Collecting the result polygons
The final step, after all bridge edges have been inserted, is collecting the resulting polygons. For that purpose I use the `Visited` attribute in the edge data structure to indicate if an edge has been visited already and therefore, has been assigned already to one of the result polygons.