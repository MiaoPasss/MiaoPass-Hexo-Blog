---
title: OpenGL顶点绘制数据
date: 2025-02-27 11:30:52
categories: Rendering
tags:
- OpenGL
---

* VAO（vertex-array object）顶点数组对象，用来管理VBO。
* VBO（vertex buffer object）顶点缓冲对象，用来缓存用户传入的顶点数据。
* EBO（element buffer object）索引缓冲对象，用来存放顶点索引数据。

A VAO is an array of VBOs

![](VAO.png "VAO和VBO之间的关系") <br>

# 一、什么是OpenGL
OpenGL是一套方便于用户使用的规范，而其本身包含了调用不同厂商直接在GPU中写好的程序接口，那些接口完成所有的功能实现，如完成2D、3D矢量图形渲染等功能（跨语言，跨平台）。

OpenGL by itself is not an API, but merely a specification. The OpenGL specification specifies exactly what the result/output of each function should be and how it should perform. It is then up to the developers implementing this specification to come up with a solution of how this function should operate. 
The people developing the actual OpenGL libraries are usually the graphics card manufacturers. Each graphics card that you buy supports specific versions of OpenGL which are the versions of OpenGL developed specifically for that card (series).

## OpenGL vs OpenCL
1. 总体来说，OpenGL 主要做图形渲染，OpenCL 主要用 GPU 做通用计算。图形渲染的主要特点是 渲染管线基本单元，如光栅化、深度测试、模板测试、混合等等的实现。
2. OpenGL 可以用 Compute Shader 实现 OpenCL 同样的功能，但一般厂商对 Compute Shader 中低精度计算的支持（fp16 / int8 等）不如 OpenCL ，性能会差一些
3. 基于 OpenCL 编程可以自己实现 OpenGL 中的渲染操作，但由于没有图形接口，实现渲染管线基本单元效率较低。
4. OpenCL 和 OpenGL 最终实现都是往 GPU 发 Command Buffer ，不会互相影响，最多就是互相抢GPU计算资源。但如果有数据依赖关系，因为管线不同，两者是需要做额外同步的。

opengl里叫drawcall，opencl里叫enqueue，vulkan里叫commandbuffer，虽然叫法不一样，但目的都是把指令从CPU发到GPU上运算。

cuda其实就是把kernel代码和c代码混合到一起写而已，最终也要把数据发到GPU去算，属于编译器支持的隐式发送。就好比写c/c++调其他库api的时候，可以配置好库lib或者so位置，包个头文件就直接用，也可以自己动态加载对应库API一样。


## OpenGL渲染过程
因为c++写的程序都是在cpu上运行的，但是OpenGL的接口是在GPU上运行的，而且OpenGL并不能凭空做程序中的数据或者是取代一些程序上的事，它是一种状态机，程序从cpu发数据到缓冲区并且告诉GPU你从哪一块缓冲区取数据画什么，然后提前设计好的着色器开始根据数据画图最后显示在显示器上。

画图渲染的顺序如下：
1. 声明一个缓冲区
2. 声明之后需要绑定，因为在GPU中的缓冲区都是有编号的，或者说是有管理的
3. 现在要给一个缓冲区塞数据，每个接口函数都可以通过说明文档来查看参数的意义和使用
4. 我们需要告诉着色器我们的数据是怎么样的， 或者说是怎么处理这些数据

## glVertexAttribPointer函数
```c++
void glVertexAttribPointer(GLuint index, GLint size, GLenum type,
    GLboolean normalized, GLsizei stride, const GLvoid * pointer);
```

![](glVertexAttribPointer.png "glVertexAttribPointer函数参数解释")

人话就是：

index：我们从第几个顶点开始访问
size：一个顶点属性值里面有几个数值
type：每个值的数据类型
normalized：是否要转化为统一的值
stride：步幅 每个顶点属性值的大小，就是到下一个顶点的开始的字节偏移量。
pointer：在开始访问到顶点属性值的时候开始的指针位置（注意和Index的区别）
其实你就是把顶点属性值想象成结构体就行了，然后多个结构体一起存，和网络传输一样，我发送给了另一边需要解析网络包，是不是需要找我结构体开始的位置，然后一个结构体的大小，然后结构体对齐里面有什么，分别解析，还有步幅指针，我还可以跳过结构体，是一样的道理。

# 二、VBO

## glVertex
最原始的设置顶点方法，在glBegin和glEnd之间使用。OpenGL3.0已经废弃此方法。每个glVertex与GPU进行一次通信，十分低效。
```c++
glBegin(GL_TRIANGLES);
    glVertex(0, 0);
    glVertex(1, 1);
    glVertex(2, 2);
glEnd();
```

## 顶点数组 Vertex Array
顶点数组也是收集好所有的顶点，一次性发送给GPU。不过数据不是存储于GPU中的，绘制速度上没有显示列表快，优点是可以修改数据。
```c++
#define MEDIUM_STARS 40
M3DVector2f vMediumStars[MEDIUM_STARS];
//在这做点vMediumStars的设置//
glVertexPointer(2, GL_FLOAT, 0, vMediumStars);
glDrawArrays(GL_POINTS, 0, MEDIUM_STARS);
```

## VBO 顶点缓冲对象
VBO，全称为Vertex Buffer Object，与FBO，PBO并称，但它实际上老不少。就某种意义来说，他就是VA（Vertex Array）的升级版。VBO出现的背景是人们发现VA和显示列表还有让人不满足的地方。一般，在OpenGL里，提高顶点绘制的办法：

(1)显示列表：把常规的glBegin()-glEnd()中的代码放到一个显示列表中（通常在初始化阶段完成），然后每遍渲染都调用这个显示列表。
(2)VA：使用顶点数组，把顶点以及顶点属性数据作为数组，渲染的时候直接用一个或几个函数调动这些数组里的数据进行绘制，形式上是减少函数调用的次数（告别glVertex），提高绘制效率。

但是，这两种方法都有缺点。VA是在客户端设置的，所以执行这类函数（glDrawArray或glDrawElement）后，客户端还得把得到的顶点数据向服务端传输一次（所谓的“二次处理”），这样一来就有了不必要的动作了，降低了效率——如果我们写的函数能直接把顶点数据发送给服务端就好了——这正是VBO的特性之一。显示列表的缺点在于它的古板，一旦设定就不容许修改，所以它只适合对一些“固定”的东西的绘制进行包装。（我们无办法直接在硬件层改顶点数据，因为这是脱离了流水线的事物）。
而VBO直接把顶点数据交到流水线的第一步，与显示列表的效率还是有差距，但它这样就得到了操作数据的弹性——渲染阶段，我们的VBO绘制函数持续把顶点数据交给流水线，在某一刻我们可以把该帧到达了流水线的顶点数据取回客户端修改(Vertex mapping)，再提交回流水线(Vertex unmapping)，或者用 glBufferData/glBufferSubData 重新全部或buffer提交修改了的顶点数据，这是VBO的另一个特性。

VBO结合了VA和显示列表这个说法不太妥当，应该说它结合了两者的一些特性，绘制效率在两者之间，且拥有良好的数据更改弹性。这种折衷造就了它一直为目前最高的地位。

当我们在顶点着色器中把顶点传送到GPU中的时候，不能每个顶点数据传送一次，因为太耗时而且造成资源浪费，所以就要用到缓冲对象，我们把大量的数据存储在GPU内存上，然后一次传输大量数据到显卡上，顶点缓冲对象就是帮助我们来管理GPU内存的。

## 顶点缓冲流程
首先我们需要使用glVertexAttribPointer函数告诉OpenGL该如何解析顶点数据，一定要使用该函数来配置各个属性的数据，因为顶点数据不只包含位置，还可能会包含顶点颜色、顶点法线等等，那一个顶点数据是如何被OpenGL解析然后放入到顶点着色器的各个属性中，就需要通过该函数进行准确的配置。
每个顶点属性从一个VBO管理的内存中获得它的数据，而具体是从哪个VBO（程序中可以有多个VBO）获取则是通过在调用glVetexAttribPointer时绑定到GL_ARRAY_BUFFER的VBO决定的，因为同一个类型的缓冲区同时最多绑定一个目标缓冲。

# 三、VAO

## VAO 顶点数组对象
VAO并不是必须的，VBO可以独立使用，VBO缓存了数据，而数据的使用 方式（glVertexAttribPointer 指定的数据宽度等信息）并没有缓存，VBO将顶点信息放到GPU中，GPU在渲染时去缓存中取数据，二者中间的桥梁是GL-Context。GL-Context整个程序一般只有一个，所以如果一个渲染流程里有两份不同的绘制代码，当切换VBO时（有多个VBO时，通过glBindBuffer切换 ），数据使用方式信息就丢失了。而GL-context就负责在他们之间进行切换。这也是为什么要在渲染过程中，在每份绘制代码之中会有glBindbuffer、glEnableVertexAttribArray、glVertexAttribPointer。VAO记录该次绘制所需要的所有VBO所需信息，把它保存到VBO特定位置，绘制的时候直接在这个位置取信息绘制。　

VAO的全名是Vertex Array Object，首先，它不是Buffer-Object，所以不用作存储数据；其次，它针对“顶点”而言，也就是说它跟“顶点的绘制”息息相关。我们每一次绘制的时候，都需要绑定缓冲对象以此来拿到顶点数据，都需要去配置顶点属性指针以便OpenGL知道如何来解析顶点数据，这是相当麻烦的，对一个多边形而言，它每次的配置都是相同的，如何来存储这个相同的配置呢。
VAO为我们解决了这个大麻烦，当配置顶点属性数据的时候，只需要将配置函数调用执行一次，随后再绘制该物体的时候就只需要绑定相应的VAO即可，这样，我们就可以通过绑定不同的VAO（提醒，与VBO一样，同一时刻只能绑定一个VAO），使得在不同的顶点数据和属性配置切换变得非常简单。VAO记录的是一次绘制中所需要的信息，这包括“数据在哪里glBindBuffer”、“数据的格式是怎么样的glVertexAttribPointer”、shader-attribute的location的启用glEnableVertexAttribArray。

```
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
```

VAO的使用非常简单，要想使用VAO，要做的只是使用glBindVertexArray绑定VAO。从绑定之后起，我们应该绑定和配置对应的VBO和属性指针，配置完以后，VAO中就存储了我们想要的各种数据，之后解绑VAO供之后使用，再次使用需要我们再次绑定。

注意：glVertexAttribPointer()这个函数会自动从当前绑定的VBO里获取顶点数据，所以在第一节VBO里如果想要使用正确的顶点数据，每次都要绑定相应的VBO，但是现在，我们在绑定VBO之前绑定了VAO，那么glEnableVertexAttribPointer()所进行的配置就会保存在VAO中。我们可以通过不同的配置从一个VBO内拿到不同的数据放入不同的VAO中，这样VAO中就有了我们所需要的数据，它根据顶点配置到VBO中索取到数据，之后直接绑定相应的VAO即可，glDrawArrays()函数就是需要从VAO中拿到数据进行绘制。但是要明白的是，我们是通过VAO来间接绑定VBO的，实际上数据还是要存储在VBO中的，VAO内并没有存储顶点的数据，如果我们要绘制两个不同的三角形，我们可能需要定义两个VBO，每个VBO内有三个顶点数据。


---

# Credits

OpenGL: https://learnopengl.com/Getting-started/OpenGL
VAO和VBO: https://blog.csdn.net/p942005405/article/details/103770259

---