**In this article, we explore state-of-the-art point cloud rendering techniques and show how we built our custom compute-based render pipeline.  
在这篇文章中，我们探讨了最先进的点云渲染技术，并展示了我们如何构建自定义的基于计算的渲染管线。**

We were tasked with rendering very large point cloud data sets, measured in the order of millions of points. Not only did this have to run at interactive frame rates, but it needed to look good in scenes at eye level. This meant we needed some form of point rejection to avoid rendering points that should be occluded. We achieved this by implementing state-of-the-art whitepapers and some Magnopus magic to get it all working in the Unity game engine.  
我们的任务是渲染非常庞大的点云数据集，数量达到百万级点。不仅需要以交互式帧率运行，而且需要在眼睛水平的场景中看起来很好。这意味着我们需要某种形式的点拒绝，以避免渲染应该被遮挡的点。我们通过实施最先进的白皮书和一些Magnopus魔法来实现这一点，使其在Unity游戏引擎中全部运行。

<iframe width="200" height="113" src="https://www.youtube.com/embed/2ci_dPrewN0?feature=oembed&amp;enablejsapi=1" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen="" title="How We Render Extremely Large Point Clouds: Cloud Zoom" id="yui_3_17_2_1_1706599060232_103" data-gtm-yt-inspected-10="true"></iframe>

### The Problem 问题

First of all, why do we need this tech? After all, there are a couple of point cloud plugins on the Unity Asset Store that render point clouds and [one in particular](https://assetstore.unity.com/packages/tools/utilities/point-cloud-viewer-and-tools-16019) that touts the ability to render datasets of billions of points. Unreal even has [one built-in to the engine](https://docs.unrealengine.com/4.27/en-US/WorkingWithContent/LidarPointCloudPlugin/) that’s free to use. The reason essentially boils down to the fact that these tools are generally meant for visualizing datasets without prioritizing visual quality. That’s not to say that any of these plugins do a bad job at rendering the point clouds; there are just some tradeoffs that these techniques leverage that we would rather not make.  
首先，我们为什么需要这项技术呢？毕竟，Unity Asset Store上有几个点云插件可以渲染点云，其中一个插件宣称可以渲染数十亿个点的数据集。Unreal甚至在引擎内置了一个免费使用的点云插件。基本上，原因归结为这些工具通常用于可视化数据集，而不是优先考虑视觉质量。这并不是说这些插件在渲染点云时做得不好；只是这些技术存在一些我们宁愿不要做出的折衷。

### The Status Quo 现状

The simplest approach to rendering point clouds is leveraging the point rasterization pipeline present in many rendering APIs. On the surface, you may think that since the APIs have a first-class rendering mode for points, it would be the fastest way to render point data to the screen. However, once you delve into the APIs rasterization rules, you will find that [under the hood](https://learn.microsoft.com/en-us/windows/win32/direct3d11/d3d10-graphics-programming-guide-rasterizer-stage-rules#point-rasterization-rules-without-multisampling), each point is essentially expanded into a two-triangle camera-facing quad.  
渲染点云的最简单方法是利用许多渲染API中存在的点光栅化管线。表面上，你可能会认为由于API具有一流的点渲染模式，这将是将点数据快速渲染到屏幕上的最快方式。然而，一旦你深入研究API的光栅化规则，你会发现，在底层，每个点实质上被扩展为一个由两个三角形组成的面向摄像机的四边形。

The problem with this technique is that it produces very small triangles which are not optimal for modern GPU architectures. Every pixel rendered by a GPU evaluates at least four times in a 2x2 tile to compute derivatives that get used for things such as texture mipmap selection. This is something that cannot be disabled and you are paying for whether you use the derivatives or not. Depending on the specific GPU architecture, the minimum tile size may be [much larger than that](https://www.g-truc.net/post-0662.html#:~:text=However%2C%20each%20triangle%20is%20effectively%20sliced%20into%20multiple%20primitives%20within%208%20by%208%20pixel%20tiles.).  
这种技术的问题在于它生成的三角形非常小，不适合现代GPU架构。GPU渲染的每个像素至少在2x2的瓦片中评估四次，以计算用于纹理mipmap选择等目的的导数。这是无法禁用的，无论你是否使用这些导数，你都要付出代价。根据具体的GPU架构，最小的瓦片大小可能远远大于这个值。

Some point cloud renderers opt to substitute a small mesh for each point in the point cloud to give volume and artistic style to the points in the cloud. This does give more control over the look, but generally exacerbates the issues mentioned above with very small triangles.  
一些点云渲染器选择为点云中的每个点替换一个小网格，以赋予点云体积和艺术风格。这确实可以更好地控制外观，但通常会加剧上述提到的非常小三角形的问题。

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/8117832d-5896-4e89-bc4d-2456da522416/image6.png)

_The Magnopus office scan rendered with unlit pyramids in place of each point.  
Magnopus办公室扫描呈现出未点亮的金字塔。_

For most cases, this is a nonissue due to the fact that the number of points is typically few enough that modern GPUs churn through the tiny triangles with ease. However, for the case of rendering extremely large point clouds, the performance issues are very noticeable.  
对于大多数情况来说，这并不是一个问题，因为点的数量通常很少，现代GPU可以轻松处理这些小三角形。然而，对于渲染极大的点云的情况，性能问题就会非常明显。

Many point cloud renderers employ some form of level-of-detail (LOD) to reduce the number of points submitted to the GPU. The fewer points that are sent to the GPU, the faster it will be at rendering them. This also has the added benefit of reducing the amount of flickering and aliasing when the point clouds are far enough from the user. However, this usually results in holes in the point cloud at large distances despite there being plenty of data in the raw point cloud to populate all pixels in that area.  
许多点云渲染器采用某种形式的细节层次（LOD）来减少提交到GPU的点数。发送到GPU的点越少，渲染速度就越快。这也有助于减少点云在用户远离时的闪烁和混叠。然而，这通常会导致点云在较远距离出现空洞，尽管原始点云中有足够的数据来填充该区域的所有像素。

Another common technique is to take the point cloud data and transform it into another data structure so it can be rendered via some other technique. This can be running an algorithm that transforms a point cloud into a 3D mesh or voxels. There is also a lot of research in the space of 3D Gaussian splatting which is a form of point cloud rendering that can produce some very compelling images. Unfortunately, each of these techniques are lossy ways of rendering a point cloud. It’s something we are keeping an eye on, but not quite a fit for our current needs.  
另一种常见的技术是将点云数据转换为另一种数据结构，以便通过其他技术进行渲染。这可以通过运行将点云转换为3D网格或体素的算法来实现。还有很多关于3D高斯飞溅的研究，这是一种可以产生非常引人注目图像的点云渲染形式。不幸的是，这些技术都是渲染点云的有损方式。这是我们正在关注的问题，但目前并不完全符合我们的需求。

### The Solution 解决方案

The graphics pipeline is highly optimized for triangulated meshes, and as we have seen above, shoehorning point cloud rendering into that pipeline has resulted in some inefficiencies. The solution is to ignore the graphics pipeline completely.  
图形管线针对三角网格进行了高度优化，正如我们在上面所看到的，将点云渲染强行塞入该管线中导致了一些效率低下的问题。解决方案是完全忽略图形管线。

Instead, we built an entirely new rasterization pipeline via the GPU’s compute shader pipeline. Compute shaders enable general-purpose computing on the GPU and as a result, there are no assumptions made about the data coming in and out of the pipeline avoiding the inefficiencies mentioned in the above techniques.  
相反，我们通过GPU的计算着色器管线构建了全新的光栅化管线。计算着色器使GPU上的通用计算成为可能，因此对管线中输入和输出的数据不做任何假设，避免了上述技术中提到的低效率。

Four whitepapers are at the heart of our point cloud rendering technique:  
我们的点云渲染技术的核心是四篇白皮书：

-   [Rendering Point Clouds with Compute Shaders and Vertex Order Optimization](https://www.cg.tuwien.ac.at/research/publications/2021/SCHUETZ-2021-PCC/) for efficiently writing the cloud data to the screen buffer.  
    使用计算着色器和顶点顺序优化渲染点云，以高效地将点云数据写入屏幕缓冲区。
    
-   [Real-time Rendering of Massive Unstructured Raw Point Clouds using Screen-space Operators](http://indigo.diginext.fr/EN/Documents/vast2011-pbr.pdf) for point rejection and hole filling.  
    使用屏幕空间运算实时渲染大规模非结构化原始点云，用于点拒绝和填补空洞。
    
-   [Real-Time Continuous Level of Detail Rendering of Point Clouds](https://www.cg.tuwien.ac.at/research/publications/2019/schuetz-2019-CLOD/) for a clever Level of Detail implementation.  
    实时连续细节级别渲染点云，用于巧妙的细节级别实现。
    
-   [Software Rasterization of 2 Billion Points in Real Time](https://www.cg.tuwien.ac.at/research/publications/2022/SCHUETZ-2022-PCC/SCHUETZ-2022-PCC-paper.pdf) for improved data throughput.  
    实时软件光栅化20亿个点，以提高数据吞吐量。
    

First, let’s explore the relevant parts of each of these papers before we talk about how we pieced them together to make our custom point cloud rasterization pipeline.  
首先，让我们在讨论如何将它们组合在一起以创建我们自定义的点云光栅化管线之前，先来探索这些论文的相关部分。

### Ordering and Rendering Points  
点的排序和渲染

The first, and most important part of the puzzle is rendering the raw point cloud data on screen. As mentioned before, we prefer to avoid the graphics pipeline. The whitepaper [Rendering Point Clouds with Compute Shaders and Vertex Order Optimization](https://www.cg.tuwien.ac.at/research/publications/2021/SCHUETZ-2021-PCC/) spells out exactly what we want to do in its title. This paper has some clever tricks that we make use of to efficiently render point clouds to the screen as well as solve aliasing problems that come with the territory of very large point clouds.  
拼图的第一部分，也是最重要的部分是在屏幕上呈现原始点云数据。如前所述，我们更喜欢避免使用图形管线。白皮书《使用计算着色器和顶点顺序优化渲染点云》在标题中准确阐明了我们想要做的事情。这篇论文中有一些巧妙的技巧，我们利用这些技巧有效地将点云渲染到屏幕上，并解决与非常大的点云相关的走样问题。

#### Reordering Points for Big Performance Gains  
重新排序点以获得更大的性能提升

<iframe width="200" height="113" src="https://www.youtube.com/embed/FTnZ1Vl-0kg?feature=oembed&amp;enablejsapi=1" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen="" title="How We Render Extremely Large Point Clouds: Cloud Sorting Order" id="yui_3_17_2_1_1706599060232_147" data-gtm-yt-inspected-10="true"></iframe>

Surprisingly, one of the biggest improvements to performance comes by simply reordering the points in the dataset. Since point clouds contain points with no connectivity information, we are free to reorder the points in the dataset. If you load a dataset in the order that a scanner has saved it out, you will usually see that neighboring points are physically far from one another. This is because the scanners typically capture a color+depth image and then write out the points in the order they are in the source image (i.e. left-to-right, top-to-bottom). Since each pixel can have a wildly different depth to them, discontinuities are unavoidable.  
令人惊讶的是，性能的最大改进之一就是简单地重新排列数据集中的点。由于点云包含没有连接信息的点，我们可以自由地重新排列数据集中的点。如果按照扫描仪保存的顺序加载数据集，通常会发现相邻的点在物理上相距很远。这是因为扫描仪通常捕捉彩色+深度图像，然后按照它们在源图像中的顺序写出点（即从左到右，从上到下）。由于每个像素的深度可能差异很大，不连续性是不可避免的。

To solve this, the paper suggests sorting the points using Morton ordering and shuffling contiguous blocks of points around. Morton ordering is also known as Z-ordering. It’s a way of ordering three-dimensional points in a one-dimensional array so that each point in the array is adjacent to the next closest point in three-dimensional space. This ensures that when the GPU launches a wave of threads, the points within the wave are projected onto a small two-dimensional area on the screen which in turn reduces the chances of expensive cache misses.  
为了解决这个问题，该论文建议使用莫顿排序对点进行排序，并对相邻的点块进行洗牌。莫顿排序也被称为Z排序。这是一种将三维点按顺序排列在一维数组中的方法，以便数组中的每个点都与三维空间中最接近的下一个点相邻。这确保了当GPU启动一波线程时，波内的点被投影到屏幕上的一个小的二维区域，从而减少了昂贵的缓存未命中的机会。

The next step is to shuffle blocks of points within the dataset. This may sound counterintuitive at first since we just neatly organized our dataset. But the reason for the shuffling is that when the data is sorted perfectly, you increase the chance that two or more waves that are in flight at the same time map points that occupy the same screen pixel. Since only one thread can safely write to a memory address at the same time, the other threads will stall out waiting for their turn to write their data to the buffer. By shuffling the points in blocks at a minimum of a wave’s execution size, we retain order within the wave but decrease the chance that multiple waves are fighting over the same pixel.  
下一步是在数据集内部对点块进行洗牌。这一开始可能听起来有些违反直觉，因为我们刚刚整理好了数据集。但洗牌的原因是，当数据被完美排序时，增加了同时在飞行中的两个或更多波浪映射到占据同一屏幕像素的点的机会。由于只有一个线程可以安全地同时写入内存地址，其他线程将因等待轮到它们写入数据到缓冲区而停滞。通过以至少一个波的执行大小对点进行块洗牌，我们在波内保持顺序，但减少了多个波争夺同一像素的机会。

#### Rendering the Points 渲染点

Now that the points are in an optimal ordering, we need to write the points to the screen somehow. The paper leverages atomic shader instructions to efficiently render the scene. There are multiple techniques that the paper explores, but the one we are interested in is what they call “high-quality shading” which is not only fast but solves the issue of aliasing.  
现在点已经以最佳顺序排列，我们需要以某种方式将点写入屏幕。该论文利用原子着色器指令高效地渲染场景。论文探讨了多种技术，但我们感兴趣的是他们所称的“高质量着色”，不仅速度快，而且解决了混叠问题。

The technique requires creating two array buffers; one for depth information, and another for color data. Each element in these buffers map to a single pixel on-screen. The depth data is stored in 32-bit unsigned integers, and color data in two 64-bit unsigned integers. The color data stores red color in the upper 32 bits, and green in the lower 32 bits of the first integer. The second integer stores blue in the upper bits, and a counter for the number of color values summed in that pixel within the lower bits.  
该技术需要创建两个数组缓冲区；一个用于深度信息，另一个用于颜色数据。这些缓冲区中的每个元素映射到屏幕上的单个像素。深度数据存储在32位无符号整数中，颜色数据存储在两个64位无符号整数中。颜色数据将红色存储在第一个整数的高32位中，将绿色存储在低32位中。第二个整数存储蓝色在高位，并在低位存储该像素中颜色值总和的计数器。

In the first pass, all points are projected onto the screen and the closest point’s depth value is stored in the buffer. Then, in the second pass, we re-project all the points, but this time store color data for each point that is within an epsilon of the recorded depth value. When a point is within the epsilon of the stored depth data, the color information is added to the color buffers and the point counter is incremented by one. This is all done with atomic add functions which are fast and avoid any race conditions when two or more points are projected to the same pixel at the same time. In a final pass, this data is then converted to color data and written to the framebuffer. The summed color data is divided by the number of points that were accumulated for each pixel giving the final color.  
在第一次遍历中，所有点都被投影到屏幕上，并且最接近的点的深度值被存储在缓冲区中。然后，在第二次遍历中，我们重新投影所有的点，但这次为每个在记录的深度值附近的点存储颜色数据。当一个点在存储的深度数据的epsilon范围内时，颜色信息被添加到颜色缓冲区，并且点计数器增加一。所有这些都是通过原子加函数完成的，这些函数快速且避免了当两个或更多点同时投影到同一个像素时的竞争条件。在最后一次遍历中，这些数据被转换为颜色数据并写入帧缓冲区。累积的颜色数据被除以每个像素累积的点数，得到最终的颜色。

### Point Rejection and Hole Filling  
点拒绝和孔填充

Once we have the points written to the screen, we need to fill any gaps between the points efficiently. Because the points are infinitesimally small and map to a single pixel, we will often see points of distant objects rendered between points that are far closer to the camera. Visually this is confusing. You would expect the object closer to the camera to occlude anything behind it, which is why we need to reject or remove any points that should be occluded before running the hole filling algorithm.  
一旦我们将点写入屏幕，我们需要有效地填补点之间的任何空隙。由于这些点是无限小的，映射到单个像素，我们经常会看到远处物体的点在比靠近摄像机的点之间渲染。在视觉上这是令人困惑的。你会期望靠近摄像机的物体遮挡其后的任何东西，这就是为什么在运行填充算法之前我们需要拒绝或移除任何应该被遮挡的点。

Take the following screenshot for example:  
以以下截图为例：

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/28c37d4d-843c-454e-885d-a5f88fb4c790/image10.png)

There is the top of a chair in the foreground, but we can see straight through it to the desks behind it which is not correct.  
前景中有一把椅子的顶部，但我们可以直接看到它后面的桌子，这是不正确的。

Now let’s take a look at what the scene looks like with point rejection enabled:  
现在让我们看看启用点拒绝后场景的样子：

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/96d20c8a-4557-443f-9719-63e1a91d0530/image13.png)

The top of the chair is much more visible in this shot and the desks behind are fully occluded. Unfortunately, this now creates a problem where we have a lot of empty space between the points.  
这张照片中椅子的顶部更加显眼，后面的桌子完全被遮挡住了。不幸的是，这现在造成了一个问题，我们在这些点之间有很多空白的空间。

The hole filling algorithm expands each of the points that remain until the empty space is filled in. Here is what the final render of the scene looks like:  
孔填充算法会扩展每个剩余的点，直到空白空间填满。以下是场景的最终渲染效果：

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/d7c4a09f-6df3-42b4-9716-02af12fa1ecd/image5.png)

The chair finally looks solid and occludes the scene behind it, just as we would expect to see.  
椅子最终看起来坚固，挡住了它后面的场景，就像我们期望看到的那样。

The paper [Real-time Rendering of Massive Unstructured Raw Point Clouds using Screen-space Operators](http://indigo.diginext.fr/EN/Documents/vast2011-pbr.pdf) details how to reject distant and occluded points, as well as fill the resulting holes.  
《实时使用屏幕空间运算渲染海量非结构化原始点云》一文详细介绍了如何拒绝远距离和遮挡点，以及填补由此产生的空洞。

To reject points, each point in the image is evaluated against other nearby points in a 7x7 grid where the evaluated pixel is directly in the center. For each point in the grid that is closer to the camera, a shader calculates the largest cone about the center that can exist without intersecting any of the other points. Then, if this cone is smaller than a defined threshold, the point is determined to be occluded and is removed from the buffer.  
拒绝点，图像中的每个点都会与周围的其他点进行评估，这些点位于一个7x7的网格中，被评估的像素直接位于中心。对于网格中更接近摄像机的每个点，着色器会计算关于中心的最大锥体，该锥体可以存在而不与其他点相交。然后，如果这个锥体小于一个定义的阈值，那么该点被确定为被遮挡，并从缓冲区中移除。

To fill the resulting holes, a recursive point expansion algorithm is run. With each iteration, every empty pixel that has neighboring pixels with color data updates its color to the average of those neighbors. The same is done with the depth information. The kernel is run three times to expand a point as far as the 7x7 bounds used in the rejection pass.  
为填补产生的空洞，运行递归点扩展算法。每次迭代时，具有颜色数据的相邻像素的每个空像素都会更新其颜色为邻居的平均值。深度信息也是如此。内核运行三次，以将点扩展到在拒绝通道中使用的7x7边界。

### Level of Detail 细节级别

When we deal with very dense point clouds, there will be times when many points project to a single point on screen. This becomes a large bottleneck, because our rendering logic uses atomic operations and many threads become blocked while they wait their turn to write data to the pixel. While the points will average together nicely with the antialiasing logic, we quickly get a diminishing return on our efforts, especially considering the performance cost. The solution here is to reduce the number of points in a way that doesn’t introduce more holes or negatively affect the frame.  
当我们处理非常密集的点云时，会有很多点投影到屏幕上的同一个点。这会成为一个很大的瓶颈，因为我们的渲染逻辑使用原子操作，许多线程在等待写入像素数据时会被阻塞。虽然点会通过抗锯齿逻辑很好地平均在一起，但我们很快就会看到我们的努力带来的回报在迅速减少，尤其是考虑到性能成本。解决方案是以一种不会引入更多空洞或对帧产生负面影响的方式减少点的数量。

Typically, other point cloud renderers use a tile-based LOD system. This would reduce the number of points so that data contention is not an issue, but because the density can only be reduced by the resolution of a tile, you will often get holes appearing in the cloud as well as visual pops.  
通常，其他点云渲染器使用基于瓦片的LOD系统。这样可以减少点的数量，以避免数据争用，但由于密度只能通过瓦片的分辨率来减少，因此您经常会看到云中出现空洞和视觉跳变。

The paper [Real-Time Continuous Level of Detail Rendering of Point Clouds](https://www.cg.tuwien.ac.at/research/publications/2019/schuetz-2019-CLOD/) presents a new LOD system that is specific to point clouds which it calls Continuous Level of Detail, or CLOD for short.  
该论文介绍了一种针对点云的新LOD系统，称为连续细节级别渲染（CLOD）。

Since point cloud points have no connectivity data, we can easily discard a point if we determine that it does not make any meaningful difference in the final frame. Therefore we can operate at the granularity of a single point for LOD generation which is exactly what the CLOD technique does. This eliminates visual pops as well as minimizing the number of holes created.  
由于点云点没有连接数据，如果我们确定它在最终帧中没有任何实质性的差异，我们可以轻松地丢弃一个点。因此，我们可以在LOD生成的粒度上操作单个点，这正是CLOD技术所做的。这样可以消除视觉突变，同时最小化产生的空洞数量。

Each point in the cloud is assigned an LOD value of 0 through 3. The vast majority of points are usually categorized under level 3 which is the full detail cloud. Then, every 1 meter in each dimension, a single point is chosen to be categorized in LOD 0. The same is done for every 0.5 meter with LOD 1, and 0.25 meter for LOD 2. The points in LOD 0 through 2 are proxy points that are representative of the points that exist in their respective volumes. For instance, a point of LOD 0 represents all points in the 1x1x1 meter volume around it. We also average the color of all of the points in the respective volume so there are no color aliasing issues.  
云中的每个点都被分配了一个从0到3的LOD值。绝大多数点通常被归类为级别3，即完整细节的云。然后，每个维度的每1米选择一个点被归类为LOD 0。每0.5米选择一个点被归类为LOD 1，每0.25米选择一个点被归类为LOD 2。LOD 0到2的点是代表它们各自体积中存在的点的代理点。例如，LOD 0的点代表其周围1x1x1米体积中的所有点。我们还对各自体积中所有点的颜色进行平均，以避免颜色混叠问题。

Both screenshots below are of the exact same view.The left view shows the full point cloud dataset with colors visualized, and the right view shows the LOD points that were selected. In this debug view, red is used for LOD 0, green for LOD 1, and Blue for LOD 2. LOD3 is not rendered since it is all of the other remaining points in the dataset. Notice that the LOD points are more uniformly distributed and the density varies between LOD levels as you would expect.  
下面的两个屏幕截图显示的是完全相同的视图。左侧视图显示了带有颜色可视化的完整点云数据集，右侧视图显示了所选的LOD点。在这个调试视图中，红色用于LOD 0，绿色用于LOD 1，蓝色用于LOD 2。由于LOD3是数据集中的所有其他剩余点，因此不会呈现。请注意，LOD点更加均匀分布，密度在LOD级别之间变化，正如您所期望的那样。

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/69fa5797-4907-49ca-a751-04033ad643ba/image11.png)

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/c88e699e-cbcc-46aa-bbbb-fa47413b8894/image1.png)

LOD visualization of the garage point cloud.  
车库点云的 LOD 可视化。

Then, at runtime, we iterate over all points in the cloud to evaluate if they should be visible, and copy the visible points into a new point cloud that is then rendered. A benefit is that we can amortize the cost of generating this reduced point cloud over multiple frames. This comes at the cost of empty space at the edges of the screen if the camera moves too quickly.  
然后，在运行时，我们遍历云中的所有点，以评估它们是否应该可见，并将可见点复制到一个新的点云中，然后进行渲染。一个好处是我们可以分摊生成这个减少的点云的成本，跨多个帧。这样做的代价是，如果摄像机移动得太快，屏幕边缘会出现空白。

To evaluate if a point is visible, we first do a frustum check, and if it passes that, we test the point’s distance from the camera to the LOD level assigned to the point. Since our LOD levels are coarsely organized between four levels, we would see a harsh line in the point cloud where the distance threshold between LOD levels exists. To avoid this, we assign a randomized and normalized weight for each point to smoothly blend between each LOD. This works beautifully and when we combine that with our hole filling technique, it becomes a practically invisible transition.  
要评估一个点是否可见，我们首先进行视锥体检查，如果通过了检查，我们就测试点到分配给该点的LOD级别的相机的距离。由于我们的LOD级别粗略地分为四个级别，我们会在点云中看到一个严格的线，这是LOD级别之间的距离阈值存在的原因。为了避免这种情况，我们为每个点分配一个随机化和归一化的权重，以便在每个LOD级别之间平滑过渡。这种方法效果很好，当我们将其与我们的填洞技术结合起来时，它就成为了一个几乎看不见的过渡。

### Optimizing Memory Throughput  
优化内存吞吐量

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/ca6594d1-2861-4969-8055-ea134d387017/image17.png)

The Magnopus point cloud dataset with a batch visualization mode.  
Magnopus点云数据集具有批量可视化模式。

Our final paper, [Software Rasterization of 2 Billion Points in Real Time](https://www.cg.tuwien.ac.at/research/publications/2022/SCHUETZ-2022-PCC/SCHUETZ-2022-PCC-paper.pdf), outlines how we can improve the GPU memory throughput.  
我们的最终论文《实时软件光栅化20亿点》，概述了我们如何改进GPU内存吞吐量。

It’s no secret that point clouds are memory intensive. We’re storing information for millions of points, and each point has position and color data associated with it. This can quickly balloon into multiples of gigabytes for a modestly sized point cloud if not addressed properly.  
众所周知，点云占用大量内存。我们需要存储数百万个点的信息，每个点都有与之相关的位置和颜色数据。如果不正确处理，即使是规模适中的点云也可能迅速膨胀成数千兆字节的数据。

Fortunately for us, in recent years, memory has become plentiful and cheap, so storing this data isn’t as daunting as it may seem. However, while memory has increased, the speed at which it can be accessed (also known as data throughput) hasn’t kept up. This has become a large bottleneck in rendering performance.  
幸运的是，近年来，内存变得丰富而廉价，因此存储这些数据并不像看起来那么令人畏惧。然而，虽然内存增加了，但它的访问速度（也称为数据吞吐量）并没有跟上。这已成为渲染性能的一个重大瓶颈。

The paper presents a method of organizing the points into blocks of equal size (10,240 points) and physical locality. A new buffer is created with the metadata of the axis-aligned bounding box for all of the points as well as what range of points in the point cloud it relates to. This enables two things; efficient culling of large blocks of points outside of the frustum, and informs us how much detail can be resolved in the area when projected onto the screen.  
该论文提出了一种将点组织成相等大小的块（10,240个点）和物理位置的方法。创建了一个新的缓冲区，其中包含所有点的轴对齐边界框的元数据，以及它与点云中的哪个点范围相关。这使得两件事情成为可能：在视锥体外高效剔除大块点，并告诉我们在屏幕投影时可以解析多少细节。

Knowing how large that block of points is on screen enables the next part of what makes this paper so great. We are now able to reduce the number of bytes that represent the positions of each point greatly reducing the amount of data passed around the GPU.  
知道屏幕上点块有多大使得这篇论文更加出色。现在我们能够大大减少表示每个点位置的字节数，从而大大减少在GPU周围传递的数据量。

Traditionally we store point cloud location data with a floating point value for the x, y, and z coordinates. Floating point values give us the flexibility to place our points anywhere in the world we like with little thought to the backing data. However, since it is ultimately just binary data, you cannot represent any conceivable number since that would require an infinite number of bits. Floating point values are designed so that most of the precision exists around the value 1.0 and becomes less precise the closer the number is to zero or infinity. Furthermore, all bits in a floating point number need to be present to make sense of it.  
传统上，我们使用浮点值来存储点云的位置数据，包括x、y和z坐标。浮点值使我们能够在世界上任何地方放置点，而不需要考虑后台数据。然而，由于它最终只是二进制数据，所以无法表示任何可想象的数字，因为那将需要无限数量的位。浮点值的设计使大部分精度存在于值1.0附近，并且随着数字接近零或无穷大，精度变得不那么准确。此外，浮点数中的所有位都需要存在才能理解它。

Another way we can store the point cloud positional data is in fixed-point notation. That means that each increment we make in the binary value, the number increases by a fixed amount. For example, a binary value of 21 with a fixed point resolution of 0.25 would evaluate to the number 5.25. Additionally, we can split the binary representation up and still make some sense of what the number is which this paper takes advantage of.  
我们可以以定点表示法存储点云位置数据。这意味着每次在二进制值中进行增量时，数字都会以固定量增加。例如，具有0.25的固定点分辨率的二进制值为21将计算为数字5.25。此外，我们可以拆分二进制表示，并仍然能够理解数字的含义，这正是本文利用的地方。

Each point’s position is converted into fixed-point notation to a resolution of 30 bits. The position stored in these 30 bits expresses a normalized location between the bounds of the axis-aligned bounding box that contains the point. The fixed-point number is then broken into 3 segments of 10 bits. The segments are then reorganized so that the top 10 bits of the x, y, and z coordinates are packed into a 32 bit unsigned integer with two bytes of padding. The same is done for the middle and lower bits. Then three arrays are created that store the high, medium, and low precision data respectively. We will refer to this technique here on out as “quantized batches”.  
每个点的位置被转换为30位的定点表示。存储在这30位中的位置表示了点所在的轴对齐边界框之间的归一化位置。然后将定点数分成3个10位的段。然后重新组织这些段，使得x、y和z坐标的前10位被打包成一个32位的无符号整数，再加上两个字节的填充。中间和低位也是如此。然后分别创建三个数组来存储高、中、低精度的数据。我们将在此之后称这种技术为“量化批处理”。

Now, when we evaluate each block of points, we project the bounding box into clip space to know if we need 10, 20, or 30 bits to reconstruct the world position at the resolution of the screen the points map to. Here you can see a visualization of this where high precision points are colored red, medium green, and low blue.  
现在，当我们评估每个点块时，我们将边界框投影到裁剪空间中，以确定我们需要10、20或30位来重建点映射到屏幕分辨率的世界位置。在这里，您可以看到高精度点以红色显示，中等精度点以绿色显示，低精度点以蓝色显示的可视化效果。

<iframe width="200" height="113" src="https://www.youtube.com/embed/GtjEk8nIbnc?feature=oembed&amp;enablejsapi=1" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen="" title="How We Render Extremely Large Point Clouds: Garage Quantized Batch Visualization" id="yui_3_17_2_1_1706599060232_188" data-gtm-yt-inspected-10="true"></iframe>

### Putting it All Together 整合一切

Hopefully, by now you can see that we have all the components to make an efficient point cloud renderer. We can dynamically generate an LOD, project the points onto the screen while minimizing the data throughput, reject points that should be occluded, and fill resulting holes. However, before we do any of that, we need to import our data.  
希望你现在能够看到，我们已经具备了所有的组件来制作一个高效的点云渲染器。我们可以动态生成LOD，将点投影到屏幕上，同时最大限度地减少数据吞吐量，拒绝应该被遮挡的点，并填补结果中的空洞。然而，在我们做任何这些之前，我们需要导入我们的数据。

#### The Import Process 进口流程

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/15779946-07c7-4396-a283-6ed1b792cb1a/image4.png)

We wrote a custom Unity asset importer for E57 point cloud files. A native plugin is used to parse the E57 file to retrieve all position and color data and load it into memory. We then get the componentwise min and max coordinates and store that as our axis-aligned bounding box for the entire cloud.  
我们为E57点云文件编写了自定义的Unity资产导入器。使用本地插件解析E57文件，以检索所有位置和颜色数据并将其加载到内存中。然后，我们获取分量最小和最大坐标，并将其存储为整个云的轴对齐边界框。

All points are then sorted via Z-order and shuffled. The points are split into batches of 10,240 points which is the minimum batch size suggested in our last paper. Every other batch is swapped with a batch at the end of the scan data. For instance, the second batch is swapped with the second-to-last batch. We chose this approach because it sufficiently shuffles the data while doing so in constant time.  
然后，所有点通过 Z 字顺序进行排序和洗牌。这些点被分成每批 10,240 个点的批次，这是我们上一篇论文中建议的最小批处理大小。每隔一个批次与扫描数据末尾的批次进行交换。例如，第二批次与倒数第二批次进行交换。我们选择这种方法是因为它在恒定时间内充分地洗牌数据。

Next, we identify all of our LOD points and write that data into the alpha component of the color. Since our point cloud data contains no transparency data, we use that to store the LOD metadata. To determine which point is LOD 0, 1, or 2, we keep track of which point is closest to the center of each 1x1x1 meter cube, 0.5x0.5x0.5 meter cube, and 0.25x0.25x0.25 meter cube respectively and update the metadata accordingly.  
接下来，我们识别所有的LOD点，并将该数据写入颜色的alpha分量中。由于我们的点云数据不包含透明度数据，我们利用它来存储LOD元数据。为了确定哪个点是LOD 0、1或2，我们跟踪每个1x1x1米立方体、0.5x0.5x0.5米立方体和0.25x0.25x0.25米立方体中离中心最近的点，并相应地更新元数据。

Last, we write all of the data into a custom binary file format alongside some header data.  
最后，我们将所有数据与一些头部数据一起写入自定义的二进制文件格式中。

#### Configuration 配置

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/563929b4-0825-4df3-bb12-1b89e6493847/Configuration.png)

_A view of our large point cloud rendering feature inspector in Unity.  
在Unity中查看我们的大型点云渲染功能检查器。_

We first need to prepare our point cloud rendering system by pre-processing and uploading the point cloud data to the GPU as well as creating some necessary buffers for the rendering system to operate on. Because the problem of rendering large point clouds efficiently is still unsolved, we architected this system to be highly configurable and flexible. You can choose how point clouds are represented in memory, enable or disable the continuous level of detail system, and set the aggressiveness of the point rejection logic to name a few. The system will adapt to what is configured, which means the logic outlined below may look a bit different.  
我们首先需要通过预处理和上传点云数据到GPU，以及创建一些必要的缓冲区供渲染系统操作来准备我们的点云渲染系统。由于高效渲染大型点云的问题仍未解决，我们设计了这个系统以具有高度可配置性和灵活性。您可以选择内存中点云的表示方式，启用或禁用连续细节级别系统，并设置点拒绝逻辑的侵略性等。系统将根据配置进行调整，这意味着下面概述的逻辑可能会有所不同。

I’ve decided to focus on the most interesting configuration below – a novel merging of both the continuous level of detail system as well as the quantized batches.  
我决定专注于下面最有趣的配置——将连续的细节级别系统与量化的批处理进行创新融合。

#### System Preparation 系统准备

When uploading point cloud data, we upload all of the point data in multiple buffers. The position data is stored in high, medium, and low precision uint structured buffers which store the segmented fixed-point position information. The fourth buffer is a structured buffer of uint containing color information. The red, green, and blue color data is each stored in a single byte and combined together into an unsigned integer. The remaining byte is used to store the LOD level. A fifth buffer is used to store batch information. Each entry in the batch buffer represents 10,240 points and stores information about the axis-aligned bounding box, and the offset into the point cloud where the batch starts. A batch size value is also stored. Since each batch size is a max of 10,240 points, you may think there is no need to store the size of the batch. However, this becomes useful later when we are dealing with the generated LOD point cloud.  
上传点云数据时，我们将所有点数据上传到多个缓冲区中。位置数据存储在高、中、低精度的无符号整数结构化缓冲区中，用于存储分段的定点位置信息。第四个缓冲区是包含颜色信息的无符号整数结构化缓冲区。红色、绿色和蓝色数据分别存储在单个字节中，并组合成无符号整数。剩余的字节用于存储LOD级别。第五个缓冲区用于存储批处理信息。批处理缓冲区中的每个条目表示10,240个点，并存储轴对齐边界框的信息以及批处理开始的点云偏移量。还存储了批处理大小值。由于每个批处理大小最多为10,240个点，您可能认为不需要存储批处理的大小。然而，当我们处理生成的LOD点云时，这将变得有用。

```
HLSL
struct PointCloudBatch
{
float3 Min;
uint Offset;
float3 Max;
uint Count;
};

StructuredBuffer&lt;PointCloudBatch&gt; PointCloudBatches;
StructuredBuffer&lt;uint&gt; PointCloudColors;
StructuredBuffer&lt;uint&gt; PointCloudPositionsLow;
StructuredBuffer&lt;uint&gt; PointCloudPositionsMedium;
StructuredBuffer&lt;uint&gt; PointCloudPositionsHigh;
```

We then need to create some buffers for the algorithm to use. First, we create a set of buffers that represent the LOD point cloud that we actually render to the screen. This consists of the same five buffers mentioned above and the number of elements is defined by the user which is the upper-bound of how many points can be rendered at any time. If the user also opts to amortize the LOD generation over multiple frames, we duplicate these buffers so we can double-buffer the point clouds which comes at the cost of additional memory usage.  
然后，我们需要创建一些缓冲区供算法使用。首先，我们创建一组缓冲区，表示实际渲染到屏幕的LOD点云。这包括上述提到的相同的五个缓冲区，元素的数量由用户定义，这是任何时候可以渲染的点的上限。如果用户还选择将LOD生成分摊到多个帧上，我们会复制这些缓冲区，这样我们就可以双缓冲点云，但这会增加额外的内存使用。

Lastly, we create a framebuffer attribute buffer. Each element in this buffer corresponds to a single pixel in the backbuffer and stores information about the rasterized cloud’s depth and color. This will later be copied to the framebuffer so it can be presented on screen. We need this as an intermediate buffer because a framebuffer cannot be directly used in a compute shader and we need the color data represented as uint values for the atomic operations we take advantage of in the shaders.  
最后，我们创建一个帧缓冲属性缓冲区。该缓冲区中的每个元素对应于后备缓冲区中的单个像素，并存储有关光栅化云的深度和颜色的信息。稍后将将其复制到帧缓冲区，以便在屏幕上显示。我们需要这个作为中间缓冲区，因为帧缓冲区不能直接在计算着色器中使用，我们需要将颜色数据表示为uint值，以便利用着色器中的原子操作。

#### The Custom Point Rendering Pipeline  
自定义点渲染管线

Now that we have the point clouds imported, uploaded, and intermediate buffers created, we can render it to the screen. Below is a visualization of our custom point cloud rasterization tech.  
现在我们已经导入了点云，上传了中间缓冲区，我们可以将其渲染到屏幕上。以下是我们自定义点云光栅化技术的可视化。

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/356cb9eb-b7a6-4718-aab5-3b28dadcd919/image16.png)

Our first step is to generate our LOD point cloud. A compute shader is executed that projects points into clip space and evaluates if they pass the CLOD system detailed above. Since we have our points grouped into batches, we can have each thread in the compute shader evaluate many points instead of a one-point-to-one-thread mapping. This allows us to first test the containing batch’s bounding box against the view frustum and exit the shader early if it’s not visible. This gives us a measurable improvement in the LOD generation pass that would not have been possible if we had not organized our points into batches. For the batches that are visible even slightly, we iterate over each point and only copy it when it passes the CLOD check. Before we copy the point over, we project it into world-space. This is necessary because the LOD cloud can contain points from multiple point clouds and world-space is common between them.  
我们的第一步是生成LOD点云。执行计算着色器将点投影到裁剪空间，并评估它们是否通过上面详细介绍的CLOD系统。由于我们将点分组成批次，我们可以让计算着色器中的每个线程评估多个点，而不是一对一的映射。这使我们可以首先测试包含批次的边界框是否与视锥体相交，并且如果不可见则提前退出着色器。这为LOD生成过程带来了可衡量的改进，如果我们没有将点组织成批次，这是不可能实现的。对于即使略微可见的批次，我们遍历每个点，并且只有在通过CLOD检查时才复制它。在复制点之前，我们将其投影到世界空间中。这是必要的，因为LOD点云可能包含来自多个点云的点，而世界空间是它们之间的共同空间。

Since we have millions of points to iterate over and potentially copy, this can be an expensive part of the process. Because of that, there is an option to amortize the cost over multiple frames. When this is amortized, an upper-bound is set for the number of points to evaluate in a given frame, and we keep track of the last evaluated point so we can resume in the next frame. The LOD point cloud is double-buffered so we don’t see partial culling results. Once all points have been evaluated, we swap the buffers so that the old buffer is now being written to, and the new buffer is now being presented.  
由于我们有数百万个要迭代和可能复制的点，这可能是过程中的昂贵部分。因此，有一个选项可以将成本分摊到多个帧上。当进行摊销时，为给定帧中要评估的点数设置了一个上限，并且我们跟踪上次评估的点，以便在下一个帧中恢复。LOD点云是双缓冲的，因此我们看不到部分裁剪结果。一旦所有点都被评估完毕，我们就交换缓冲区，这样旧缓冲区现在正在被写入，而新缓冲区现在正在被呈现。

Next, we render the LOD point cloud to our attribute buffer. Here is the struct we use for this buffer.  
接下来，我们将LOD点云渲染到我们的属性缓冲区。这是我们用于此缓冲区的结构体。

```
HLSL
struct PointCloudPixelAttributes
{
    uint colorCount;
    uint3 color;
    uint depth;
    uint3 pad;
};
```

For those familiar with the paper on rendering the point cloud, you will probably notice we’re not using 64-bit unsigned integers here. The reason is that 64-bit atomic operations were not introduced until shader model 6.6. Since Unity leverages HLSL, this requires the newer shader compiler DXC which was not fully integrated with the engine at the time. This is likely an area where we are leaving some performance improvements on the table, which we’ll circle back to in the future.  
对于熟悉点云渲染论文的人来说，你可能会注意到我们这里没有使用64位无符号整数。原因是直到着色器模型6.6才引入了64位原子操作。由于Unity利用HLSL，这需要较新的着色器编译器DXC，而当时它尚未完全与引擎集成。这很可能是我们在这方面留下了一些性能改进的地方，我们将在未来回过头来处理。

We first need to clear this buffer so that the previous frame’s data doesn’t corrupt our view. This is as simple as setting all the values to zero.  
我们首先需要清空这个缓冲区，以防上一帧的数据破坏我们的视图。只需将所有值设为零即可。

Now that we are working with a clean slate, we execute our depth pass which stores the closest point cloud depth value for each screen pixel. Each point is projected into clip-space by multiplying the point’s position with the view-projection matrix just as you would with any 3D object. The only difference is that we use the view-projection matrix instead of model-view-projection because the LOD point cloud’s model is stored in world-space and the model matrix would simply be the identity matrix.  
现在我们从零开始工作，执行深度通道，为每个屏幕像素存储最近的点云深度值。每个点都通过将其位置与视图投影矩阵相乘来投影到裁剪空间，就像对待任何3D对象一样。唯一的区别是，我们使用视图投影矩阵而不是模型视图投影矩阵，因为LOD点云的模型存储在世界空间中，而模型矩阵简单地是单位矩阵。

The depth value is then converted into an unsigned integer via the \`asuint\` HLSL function. Due to the way normalized floating point numbers are stored in binary, we can guarantee that any floating point number that is greater than another will also be greater when interpreted as unsigned integers. Because of that, the \`InterlockedMax\` HLSL intrinsic is used when writing the value. It will only write the value if it’s greater than what’s currently stored. This atomic function only works with unsigned integers which is why we can’t use floating point numbers directly.  
然后，深度值通过\`asuint\` HLSL函数转换为无符号整数。由于规范化浮点数在二进制中的存储方式，我们可以保证任何大于另一个的浮点数，在解释为无符号整数时也会更大。因此，在写入该值时使用了\`InterlockedMax\` HLSL内置函数。只有当该值大于当前存储的值时，它才会写入。这个原子函数只能处理无符号整数，这就是为什么我们不能直接使用浮点数。

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/b3946ba2-12fd-4825-ac64-176c0b68c8f8/image18.png)

The last step in the rasterization phase is to run the color pass. This is identical to the previous pass with a couple exceptions. First, when we test against the depth buffer, we nudge the projected point closer to the camera ever-so-slightly. Then, if it passes the depth test, we add it to our attribute’s color buffer as well as increment the color counter.  
光栅化阶段的最后一步是运行颜色通道。这与之前的通道基本相同，但有几个例外。首先，在深度缓冲区测试时，我们会轻微地将投影点靠近摄像机。然后，如果通过深度测试，我们将其添加到属性的颜色缓冲区，并增加颜色计数器。

This is where we would ideally use 64-bit atomics, but due to Unity’s limitation we stick with 32-bit atomics. We use the \`InterlockedAdd\` intrinsic for all three color channels as well as the color count variable.  
这是我们理想情况下会使用64位原子操作的地方，但由于Unity的限制，我们只能使用32位原子操作。我们对所有三个颜色通道以及颜色计数变量使用\`InterlockedAdd\`内置函数。

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/fa0ddc2f-c227-4e22-9cd7-b4689f516901/image8.png)

At this point, we could render the point cloud directly to the screen. This is actually what is happening in the above screenshot. However, we want to improve the rendered frame’s quality as much as possible. This is where our hidden surface rejection and hole filling logic comes into play.  
在这一点上，我们可以直接将点云渲染到屏幕上。这实际上就是上面截图中发生的事情。然而，我们希望尽可能提高渲染帧的质量。这就是我们的隐藏表面拒绝和填补空洞逻辑发挥作用的地方。

This part of the pipeline is more or less identical to the paper referenced above, so I won’t go into details. The gist of it is that we test each point in the attribute buffer with its neighbors to see if a neighbor is far enough closer to the camera to occlude the point. Then we iteratively expand the surviving points to any empty neighboring pixel.  
这部分管道与上述引用的论文几乎完全相同，所以我不会详细说明。其要点是，我们测试属性缓冲区中的每个点及其邻居，以查看邻居是否离摄像机足够远，以遮挡该点。然后，我们迭代地将幸存的点扩展到任何空的相邻像素。

The one difference with our implementation is that we combined the point rejection pass with the color resolve pass. A temporary texture is requested at the resolution of the backbuffer, and when a point passes the point rejection check, the color data is then divided by the number of points that were added to the buffer at that pixel and written to the intermediate buffer. By merging these two passes we save a small amount of performance.  
我们的实现与其他实现的一个区别是，我们将点拒绝检查与颜色解析合并在一起。在后台缓冲区的分辨率上请求临时纹理，当一个点通过点拒绝检查后，颜色数据将被除以在该像素处添加到缓冲区的点的数量，然后写入中间缓冲区。通过合并这两个步骤，我们可以节省一小部分性能。

Finally, we blit the temporary render texture to the framebuffer so it can be presented to the user. A depth comparison is done so that the point cloud pixels integrate with the traditional polygon geometry. This compositing is done before the transparent object rendering since transparent objects do not write to the depth buffer.  
最后，我们将临时渲染纹理传输到帧缓冲区，以便呈现给用户。进行深度比较，使点云像素与传统的多边形几何体整合。这种合成是在透明物体渲染之前进行的，因为透明物体不会写入深度缓冲区。

### The Results 结果

The largest dataset we’ve tested this with is a scan of a section of our Downtown Los Angeles office which consists of over 160 million points. This dataset is then imported into two point clouds. Due to a bug in Unity where very large data files freeze and crash the editor when selected in the project panel, we split large point clouds. The first point cloud is exactly 100 million points large, and the second is the remaining 60+ million points. The points in both clouds physically overlap which leads to some inefficiency but we didn’t pursue optimizing that.  
我们测试过的最大数据集是我们洛杉矶市中心办公室的一部分扫描，其中包含超过1.6亿个点。然后将该数据集导入两个点云中。由于Unity中存在一个bug，当在项目面板中选择非常大的数据文件时，编辑器会冻结并崩溃，因此我们将大型点云拆分。第一个点云正好包含1亿个点，第二个点云包含剩余的6000多万个点。两个点云中的点在物理上重叠，这导致了一些效率低下，但我们没有追求优化。

We found that the hybrid approach of quantized batches, mixed with the continuous level of detail, can render faster than directly rendering quantized batches in some scenarios. This comes with the tradeoff of higher memory usage, and in some cases, worse performance compared to quantized batches alone.  
我们发现，在某些情况下，将量化批次与连续的细节级别混合使用可以比直接渲染量化批次更快。这种方法的缺点是内存使用更高，在某些情况下，性能比单独使用量化批次更差。

Views 1 through 3 are all eye-level vantage points which we are mostly interested in. View 1 has the most points in view, view 2 has fewer, and view 3 has the least. Views 4 and 5 are extreme scenarios where all points are visible. View 4 keeps all points visible and generally maximized to the size of the render target. View 5 is greatly zoomed out where all of the 160+ million points converge on a small area of the screen showing issues of data contention.  
视图1到3都是我们最感兴趣的视角。视图1中有最多的点，视图2中有较少的点，视图3中有最少的点。视图4和5是极端情况，所有点都是可见的。视图4保持所有点可见，并通常最大化到渲染目标的大小。视图5是大大缩小，所有的1.6亿多个点都汇聚在屏幕的一个小区域，显示了数据争用的问题。

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/c022c07d-6bbc-4b32-83ec-875833ebc8d6/image7.png)

|  | Baseline 基线 | CLOD\* | QB | QB+CLOD | QB+CLOD Amortized QB+CLOD 分期偿还 |
| --- | --- | --- | --- | --- | --- |
| View 1 视图1 | 16.9 | 23.9 | 10.5 | 15.0 | 9.2 |
| View 2 视图2 | 15.5 | 11.9 | 7.3 | 7.7 | 4.4 |
| View 3 查看3 | 13.8 | 7.9 | 5.9 | 4.8 | 2.0 |
| View 4 查看4 | 17.8 | 26.5 | 15.1 | 23.0 | 15.0 |
| View 5 查看5 | 49.5 | 6.5 | 49.5 | 3.3 | 0.7 |

Baseline - Rendered without any level of detail or quantized batches  
基准线 - 无任何细节或量化批次渲染

CLOD - Rendered with continuous level of detail  
CLOD - 以连续细节级别渲染

QB - Rendered with quantized batches  
QB - 使用量化批次渲染

QB+CLOD - Rendered with hybrid system  
QB+CLOD - 用混合系统渲染

Timings were recorded on an NVIDIA GeForce RTX 3080 Laptop GPU.  
定时是在一台NVIDIA GeForce RTX 3080笔记本GPU上记录的。

\* - The CLOD renderer is maxed at 125 million points for the LOD point cloud which is not able to represent the entire 160+ million points in the example dataset leading to large holes in the rendered point cloud. The 125 million point limit is due to the maximum size a buffer can be allocated. This can be solved by splitting the reduced point cloud into multiple buffers but due to lack of necessity, it was not pursued.  
CLOD渲染器的LOD点云最大限制为1.25亿点，无法表示示例数据集中的1.6亿多点，导致渲染的点云中出现大洞。1.25亿点的限制是由于缓冲区的最大尺寸限制。可以通过将减少的点云分割成多个缓冲区来解决此问题，但由于缺乏必要性，未进行此项工作。

View 1 视图1

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/d310bf82-8133-49a9-a1bd-11ed5b03d9e8/image15.png)

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/20508fc6-415c-40b1-a0e1-e408cf2298be/image12.png)

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/d8b7526f-5d7a-4fd7-8e62-603dca58f90e/image9.png)

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/eaebd3d7-d343-4221-80ae-a0f834991e6e/image19.png)

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/74ac930e-9593-47be-808d-6ce55dfe9a4b/image14.png)

### Additional Findings 额外发现

#### Enhanced Hidden Surface Removal  
增强隐藏表面消除

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/7f7253c2-f51c-48fd-8a28-7d45a9f39c00/image3.png)

We explored increasing neighbor grid sizes for the point rejection pass and increasing the hole filling iteration count to have better hidden surface removal. However, this didn’t result in a better image. Increasing the neighbor grid size for point rejection causes it to be overly aggressive and starts removing points that should be visible. And increasing the hole filling iterations makes it a muddy mess. In the screenshot above, a lot of the data in the ceiling area is rejected and hole filling takes over ruining the visual quality of this frame.  
我们尝试增加邻居网格大小以进行点拒绝处理，并增加填洞迭代次数以实现更好的隐藏表面去除。然而，这并没有带来更好的图像。增加点拒绝的邻居网格大小会导致过于激进，开始移除本应可见的点。增加填洞迭代次数会使其变得混乱不堪。在上面的截图中，天花板区域的许多数据被拒绝，填洞接管并破坏了该帧的视觉质量。

#### Overdraw 透支

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/68fc11a8-3d21-43ab-9b09-a4f3069cbda3/image2.png)

As a rendering engineer I can’t have enough debug views. These help with quickly diagnosing rendering problems. An overdraw view was pretty much free to implement with the data we have in our rendering pipeline.  
作为渲染工程师，我永远不会有足够的调试视图。这些视图有助于快速诊断渲染问题。使用我们渲染管线中的数据，实现一个过度绘制视图几乎是免费的。

Overdraw is the technical term for a pixel being written to more than one time. Ideally, you would just write to a pixel once and only once. The more times a pixel is written to, the more likely performance is lost. However, Overdraw is something we leverage to solve aliasing issues inherent with point clouds. Nevertheless, it isn’t free and knowing where it’s happening is useful.  
过度绘制是指一个像素被写入超过一次的技术术语。理想情况下，你只会写入一个像素一次。一个像素被写入的次数越多，性能丢失的可能性就越大。然而，过度绘制是我们利用来解决点云固有的走样问题的。然而，这并非没有代价，了解它发生的位置是有用的。

Since our attribute buffer stores the number of points that contributed to a pixel, we can interpret it as a color. In our case, green is where a pixel was written to once, red is where it was written to 128 times or more, and the gradient of yellow/orange is some amount in between.  
由于我们的属性缓冲区存储了贡献到像素的点的数量，我们可以将其解释为颜色。在我们的情况下，绿色表示像素被写入一次，红色表示像素被写入128次或更多，而黄色/橙色的渐变则表示介于两者之间的某个数量。

### The Future 未来

![](https://images.squarespace-cdn.com/content/v1/618131cf8cd8e779321e9666/0843cada-f40a-4844-bac2-0cb775ef7754/the+future.png)

While we have been able to craft an efficient renderer for very large point clouds, we still believe that there are opportunities to improve it further.  
尽管我们已经成功地为非常大的点云创建了高效的渲染器，但我们仍然相信还有机会进一步改进它。

The first, and most obvious thing is to switch to using 64-bit unsigned integers in our algorithms. We were constrained to use 32-bit values because the version of Unity we were developing on didn’t have full DXC support, but this has likely improved in newer versions. This can reduce the number of atomic operations that occur in the point rendering logic by up to half, giving us a sizable improvement. We would expect to see all rendering techniques benefit from this as the point raster logic is shared between them all.  
首先，最明显的事情是在我们的算法中切换到使用64位无符号整数。我们之前受限于使用32位值，因为我们开发的Unity版本没有完全的DXC支持，但在新版本中这可能已经得到改善。这可以将点渲染逻辑中发生的原子操作数量减少一半，为我们带来可观的改进。我们期望看到所有渲染技术受益于此，因为点栅格逻辑在它们之间是共享的。

Also, there may be ways to optimize the hybrid point cloud reducer that we haven’t yet found and would be worth pursuing. Any improvement there would make it a more compelling technique to use over just quantized batches alone.  
此外，可能有一些优化混合点云减少器的方法，我们尚未发现，但值得追求。任何改进都将使其成为一种更具吸引力的技术，而不仅仅是使用量化批处理。

Lastly, finding an efficient way to index the attribute buffer by 2D Morton Ordering may give us improved performance gains due to improved cache coherency. Right now, the data is indexed in a linear fashion so a change in the Y-axis can result in a large jump in memory. Keeping the elements that are physically close should improve performance for everything in the pipeline before point expansion.  
最后，通过使用二维莫顿顺序对属性缓冲区进行高效索引可能会带来性能提升，因为缓存一致性得到改善。目前，数据是以线性方式索引的，因此Y轴的变化可能导致内存跳跃较大。保持物理上接近的元素应该会提高点扩展之前管道中所有内容的性能。

In conclusion, we explored why a compute-based rendering pipeline is more efficient than the graphics pipeline for point cloud rendering. We also explored the current state of the art of point cloud rendering and showed how the Magnopus team iterated on this to create a custom pipeline that even beats out the best algorithm in certain scenarios.  
总之，我们探讨了为什么基于计算的渲染管线比图形管线更适用于点云渲染。我们还探讨了点云渲染的当前技术水平，并展示了Magnopus团队如何在此基础上进行迭代，创建了一种自定义管线，甚至在某些情况下超越了最佳算法。

 [![](https://images.squarespace-cdn.com/content/v2/namespaces/memberAccountAvatars/libraries/5f2bbe2382255b3888d01cba/d0a13cd3-f3db-46ad-a7b4-f8c7c064b230/derrick.jpg?format=100w)Derrick Canfield 德里克·坎菲尔德](https://www.magnopus.com/blog?author=654ba3edb88c3254c024efa6)

Lead Rendering Engineer at Magnopus  
Magnopus的首席渲染工程师
