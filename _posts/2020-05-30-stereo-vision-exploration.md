---
layout: post
title:  "Disparity Map Computation in Python and C++"
date:   2020-05-14 06:02:41 -0400
summary: Disparity map is an essential part of Stereo Vision. This post will walk you through the implementation details with code in python and C++
use_math: true
---
Stereo vision is the term used for the process of inferring 3D depth information from 2D images [^1]. 2D images may be from a Stereo Rig, usually consisting of two cameras slightly displaced horizontally like our eyes. In fact, stereopsis [^1] takes inspiration from our ability to infer depth information using our eyes. This post is focused on implementation. There are good posts explaining all the details of stereo vision [^2] [^3]. A comprehensive and clear explanation of Stereo Vision is presented here [^4]. If you would like to understand depth calculation clearly, you can refer to [^5]. In this post, I will walk you through the implementation steps in python by referring to the stereo vision description. 

## Depth Estimation
Before we proceed coding, I would like to explain `how can we estimate depth using the disparity map?` Without this motivation, I feel it is pointless to explain disparity map implementation. I had particularly hard time understanding a specific part of the depth equation derivation which I will point out later. May be it's just me but thought I will write this up so that it may help someone who had similar question as I did. A simplified stereo setup with two cameras is shown here. To estimate depth from stereo implies that we need to estimate $Z$ in the figure. $Z$ is the distance of point $P$ from the camera.

{% include image-small.html img="images/2020-05-30/stereo_geometry.png" title="stereo_geometry" caption="Derivation of depth for a simple stereo example with two cameras with perfectly aligned image and having same focal lengths. (source: CS 4495 Computer Vision – A. Bobick)" url="https://www.cc.gatech.edu/~afb/classes/CS4495-Fall2013/slides/CS4495-06-Stereo.pdf" %}

In the above setup, let $C_{L}$ be the camera on the left and $C_{R}$ be the camera on the right. Both these cameras have the same focal length $f$. Distance between camera centers is $B$. A line from point $P$ to the camera center of $C_{L}$ intersects the image plane at $p_{l}$. A line from point $P$ to camera center of $C_{R}$ intersects $C_{R}$'s image plane at $p_{r}$. Note that the triangles $p_{l} P p_{r}$ and $C_{L}PC_{R}$ are similar triangles. Since these triangles are similar, their ratio of base to height should be the same, i.e., $\frac{B}{Z}$ = $\frac{p_{l}p_{r}}{Z-f}$. From the figure, we have $p_{l}p_{r}$ to be $B - (x_{l}+x_{r})$. However, in all the derivations in multiple references, $p_{l}p_{r}$ is told to be $B - x_{l} + x_{r}$ which totally confused me. It is quite clear from the figure, to get $p_{l}p_{r}$ we need to subtract ($x_{l} + x_{r}$) from $B$. 

Let's say I would like to test my hypothesis that $p_{l}p_{r} = B - (x_{l}+x_{r})$. When we use our depth sensing system in practice, we will have to feed in the focal length ($f$), base length ($B$), $x_{l}$, and $x_{r}$ to obtain $Z$ which is the depth estimation for point $P$. $x_{l}$ is a positive value since it is to the right of the camera center line passing through the image plane which serves as the origin. $x_{r}$ is a negative number since it is to the left of the origin. Now, if we use the equation $p_{l}p_{r} = B - (x_{l}+x_{r})$ with -ve value for $x_{r}$ we end up adding $x_{r}$ to $B$ instead of subtracting, i.e., we will end up with $p_{l}p_{r} = B - (x_{l}-x_{r}) = B - x_{l} + x_{r}$. This length is incorrect. Say, we used $p_{l}p_{r} = B - (x_{l}-x_{r})$ and since we have $x_{r}$ as negative, $p_{l}p_{r} = B - (x_{l}-(-x_{r})) = B - (x_{l}+x_{r})$. This is the reason we have $-x_{r}$ in the equation to estimate depth from stereo images. Now, a negative sign for $x_{r}$ in the depth estimation equation makes sense to me. So, finally, to estimate depth, we can use the following equation $Z=f \frac{B}{x_{l}-x_{r}}$.

## Disparity Map
Depth is inversely proportional to disparity, i.e., from the depth estimation equation, we have $Z \propto \frac{1}{x_{l}-x_{r}}$. As disparity ($x_{l}-x_{r}$) increases, $Z$ decreases and for lower disparity ($x_{l}-x_{r}$), we would have higher $Z$. This is intuitive if you hold your index finger near your eyes and alternate seeing from your left and right eyes. You will notice that your finger jumps a lot in your view compared to other distant object. We will use a technique called block matching to find the correspondences between pixels in the two images as outlined in [^2]. Here is a summary of steps we need for computing disparity map:
* Input: Left image and right image (from perfectly aligned cameras) of width $w$ and height $h$, block size in pixels, and search block size
* Output: Disparity map of width $w$ and height $h$
* Why do we need to specify block size and search block size?
    * For every pixel in the left image, we need to find the corresponding pixel in the right image. Since pixel values may be noisy and is influenced by many factors such as sensor noise, lighting, mis-alignment, etc., we may have to rely on a group of surrounding pixels for comparison. 
    * ***Block size*** refers to the neighborhood size we select to compare pixels from left image and the right image specified as number of pixels in height and width. We will use pixel similarity score to quantify the match.
    * ***Search block size*** refers to a strip of rectangle in which we will search for best matching block. Notice that in the third figure, you will have to move the white smaller rectangle to the left to get a best match of pixel similarity.
{% include image.html img="images/2020-05-30/block_matching.png" title="block_matching" caption="Block matching example. (Image source: Middlebury Stereo Datasets)" url="http://vision.middlebury.edu/stereo/data/scenes2003/" %}
* For a pixel in the left image, select the pixels in it's neighborhood specified as block size from the left image. 
* Compute similarity score for each block (same size as block size) selected from the search block and keep sliding the block by one pixel within the search block. Record all the similarity scores.
* Find the highest pixel similarity score and use the pixel at the block center as the corresponding pixel for the left pixel/block we are trying to find the best match. 
* If $x_{l}$ is the column index of the left pixel, and the highest similarity score was obtained for a block on the right image whose center is the pixel with column in index $x_{r}$, we will note the disparity value of |$x_{l}-x_{r}$| for the location of left image pixel.
* Repeat the matching process for each pixel in the left image and note all the disparity values for the left image pixel index.
* We will start building the basic building blocks first and later combine these building blocks to compute disparity map

### Similarity metric
We need to define a notion of similarity between two blocks of pixels. Sum of absolute difference between pixel values is an intuitive metric for similarity. For example, (3,5) are more similar compared to (3,6) since the absolute difference between |3 - 5| < |3 - 6|, i.e., 2 < 3. If there are multiple such values for comparison, we sum up the differences. Hence, we will implement sum of absolute difference method. We will loop over each row ($i$) and column ($j$) in both left and right blocks we are given using $\Sigma_{i,j} |B_{i,j}^{l} - B_{i,j}^{r}|$.

{% highlight python %}
import numpy as np

def sum_of_abs_diff(pixel_vals_1, pixel_vals_2):
    """
    Args:
        pixel_vals_1 (numpy.ndarray): pixel block from left image
        pixel_vals_2 (numpy.ndarray): pixel block from right image

    Returns:
        float: Sum of absolute difference between individual pixels
    """
    if pixel_vals_1.shape != pixel_vals_2.shape:
        return -1

    return np.sum(abs(pixel_vals_1 - pixel_vals_2))
{% endhighlight %}

---
[^1]: Forsyth, D., & Ponce, J. (2003). Computer vision: A modern approach. Upper Saddle River, N.J: Prentice Hall.
[^2]: http://mccormickml.com/2014/01/10/stereo-vision-tutorial-part-i/
[^3]: http://mccormickml.com/assets/StereoVision/Stereo%20Vision%20-%20Mathworks%20Example%20Article.pdf
[^4]: http://vision.deis.unibo.it/~smatt/Seminars/StereoVision.pdf
[^5]: https://www.cc.gatech.edu/~afb/classes/CS4495-Fall2013/slides/CS4495-06-Stereo.pdf