In the following you can find the source code of many implementations accompanying my blog posts. Beyond that, I uploaded my bachelor and master thesis. In one or another blog post I might write about things which are explained in more detail in one of the two theses.

# Talks
* [Beyond the Limits - The Usage of C++ in the Demoscene](http://geidav.files.wordpress.com/2014/05/beyond-the-limits-the-usage-of-cpp-in-the-demoscene.pdf): Slides for the talk I gave together with Eivind Liland at the monthly *C++ User Group Berlin* meeting in May 2014

# Theses
* [Master thesis](http://geidav.files.wordpress.com/2013/04/mscthesis.pdf) (2012): Development and evaluation of a real-time capable multiple hypothesis tracker
*Multi-object tracking is nowadays deployed in a vast number of scientific disciplines. Even though, multi-object tracking has been researched for decades it still remains a challenging problem because of its computational complexity. The aim of this master thesis is to analyze the real-time capabilities of a modern multiple hypothesis tracker, used for tracking persons in (surveillance) videos. For this purpose all required theory is discussed first and afterwards, the implementation created in the scope of this master thesis is presented. It follows an evaluation which focuses on the speed of the implementation. Potential bottlenecks are analyzed and ideas for future optimizations are given. Supplementary, the tracking quality is analyzed and compared to the one of a global nearest neighbor tracker.*

* [Bachelor thesis](http://geidav.files.wordpress.com/2013/04/bscthesis.pdf) (2009): Reconstruction of depth information from images for visualization of architectural models
*In this bachelor thesis a novel shape-from-shading technique called Depth Hallucination is presented. It's supposed to make extracting depth information from 3D surfaces as easy as taking two pictures with a digital camera: one with flash and one without. The resulting depth maps are then used to visualize the facades of architectural models. To overcome certain limitations of the Depth Hallucination technique, graph cut image segmentation is used to exclude certain regions from the hallucination process.*

# Source code
* [Splitting an arbitrary polygon by a line](https://github.com/geidav/concave-poly-splitter): Implementation of the algorithm to split an arbitrary, simple polygon by a line, described in this [blog post](https://geidav.wordpress.com/2015/03/21/splitting-an-arbitrary-polygon-by-a-line/).
* [Optimized binary search](https://github.com/geidav/lut-binary-search): Binary search implementation optimized with the look-up table technique described in this [blog post](http://geidav.wordpress.com/2013/12/29/optimizing-binary-search/).
* [Computing OMBBs in 2D](https://github.com/geidav/ombb-rotating-calipers): Implementation of the Rotating Calipers method for computing oriented minimum bounding boxes as described in this [blog post](http://geidav.wordpress.com/2014/01/23/computing-oriented-minimum-bounding-boxes-in-2d/).
* [Operator system](https://github.com/geidav/op-sys): Implementation of low-level argument passing for the operator system described in this [blog post](http://geidav.wordpress.com/2013/07/31/operator-systems-accessing-parameters-in-execute-handlers/).
