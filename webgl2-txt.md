1、栅格化(rasterization)引擎
  实际上WebGL仅仅是栅格化(rasterization)引擎。它会基于你的代码来画点，线条和三角形。 而你需要使用点、线、三角形组合来完成复杂的3D任务。

2、WebGL是在GPU上运行的。在GPU上运行的WebGL代码是以一对函数的形式，分别叫做点着色器(Vetex Shader)和片段着色器(Fragment Shader). 他们是用一种类似C++的强类型语言GLSL编写的。这一对函数组合被叫做程序(Program)。
  点着色器的任务是计算点的的位置。基于函数输出的位置，WebGL能够栅格化(rasterize)不同种类的基本元素，如点、线和三角形。当栅格化这些基本元素的同时，也会调用第二种函数：片段着色器。它的任务就是计算当前正在绘制图形的每个像素的颜色。
  几乎所有的WebGL API是为这些函数对的运行来设置状态。你需要做的是：设置一堆状态，然后调用gl.drawArrays和gl.drawElements在GPU上运行你的着色器。
  这些函数需要用到的任意数据都必须提供给GPU。 着色器有如下四种方法能够接收数据。
    1、属性(Attributes)，缓冲区(Buffers)和顶点数组(Vetex Arrays)
      缓存区以二进制数据形式的数组传给GPU。缓存区可以放任意数据，通常有位置，归一化参数，纹理坐标，顶点颜色等等
      属性用来指定数据如何从缓冲区获取并提供给顶点着色器。比如你可能将位置信息以3个32位的浮点数据存在缓存区中， 一个特定的属性包含的信息有：它来自哪个缓存区，它的数据类型(3个32位浮点数据)，在缓存区的起始偏移量，从一个位置到下一个位置有多少个字节等等。
      缓冲区并非随机访问的，而是将顶点着色器执行指定次数。每次执行时，都会从每个指定的缓冲区中提取下一个值并分配给一个属性。
      属性的状态收集到一个顶点数组对象（VAO）中，该状态作用在每个缓冲区，以及如何从这些缓冲区中提取数据。
    2、Uniforms
      Uniforms是在执行着色器程序前设置的全局变量
    3、纹理(Textures)
      纹理是能够在着色器程序中随机访问的数组数据。大多数情况下纹理存储图片数据，但它也用于包含颜色以为的数据。
    4、Varyings
      Varyings是一种从点着色器到片段着色器传递数据的方法。根据显示的内容如点，线或三角形， 顶点着色器在Varyings中设置的值，在运行片段着色器的时候会被解析。

3、WebGL只关注两件事：剪辑空间坐标(Clip space coordinates)和颜色。 所以作为程序员，你的任务是向WebGL提供这两件事--编写两种着色器的代码: 点着色器提供剪辑空间坐标；片段着色器提供颜色。
  不管你的画布大小，剪辑空间坐标的取值范围是-1到1. 

4、注意： #version 300 es 必须位于着色器代码的第一行。 它前面不允许有任何的注释或空行！ #version 300 es 的意思是你想要使用WebGL2的着色器语法:GLSL ES 3.00。 如果你没有把它放到第一行，将默认设置为GLSL ES 1.00,即WebGL1.0的语法。相比WebGL2的语法，会少很多特性。

5、在GPU上已经创建了一个GLSL程序后，我们还需要提供数据给它。大多数WebGL API是有关设置状态来供给GLSL程序数据的。 在我们的例子中，GLSL程序唯一的输入属性是a_position。我们做的第一件事就是查找这个属性的位置。记住在查找属性是在程序初始化的时候，而不是render循环的时候。
  var positionAttributeLocation = gl.getAttribLocation(program, "a_position");

6、WebGL通过绑定点来处理许多WebGL资源。你可以认为绑定点是WebGL内部的全局变量。首先你绑定一个资源到某个绑定点，然后所有方法通过这个绑定点来对这个资源的访问。下面我们来绑定缓冲区。
  gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
  现在我们通过绑定点把数据存放到缓冲区。
  // three 2d points
  var positions = [
    0, 0,
    0, 0.5,
    0.7, 0,
  ];
  gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(positions), gl.STATIC_DRAW);
  Javascript弱类型语言，而WebGL需要强类型数据，需要用new Float32Array(positions)创建32位的浮点数数组，然后用gl.bufferData函数将数组数据拷贝到GPU上的positionBuffer里面。因为前面把positionBuffer绑定到了ARRAY_BUFFER，所以我们直接使用绑定点。
  最后一个参数gl.STATIC_DRAW提示WebGL如何使用数据，WebGL据此做相应的优化。gl.STATIC_DRAW 告诉WebGL我们不太可能去改变数据的值。

7、数据存放到缓存区后，接下来需要告诉属性如何从缓冲区取出数据。首先，我需要创建属性状态集合：顶点数组对象(Vertex Array Object)。
  var vao = gl.createVertexArray();
  为了使所有属性的设置能够应用到WebGL属性状态集，我们需要绑定这个顶点数组到WebGL。
  gl.bindVertexArray(vao);
  然后，我们还需要启用属性。如果没有开启这个属性，这个属性值会是一个常量。
  gl.enableVertexAttribArray(positionAttributeLocation);
  接下来，我们需要设置属性值如何从缓存区取出数据。
  var size = 2;          // 2 components per iteration
  var type = gl.FLOAT;   // the data is 32bit floats
  var normalize = false; // don't normalize the data
  var stride = 0;        // 0 = move forward size * sizeof(type) each iteration to get the next position
  var offset = 0;        // start at the beginning of the buffer
  gl.vertexAttribPointer(
    positionAttributeLocation, size, type, normalize, stride, offset)
  gl.vertexAttribPointer 的隐含部分是它绑定当前的ARRAY_BUFFER到这个属性。换句话说，这个属性被绑定到positionBuffer。

8、通过设置gl_Position, 我们需要告诉WebGL如何从剪辑空间转换值转换到屏幕空间。 为此，我们调用gl.viewport并将其传递给画布的当前大小。
  gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);
  这行代码告诉WebGL将裁剪空间的-1~+1映射到x轴0~gl.canvas.width和y轴0~gl.canvas.height。
  我们设置画布的清空颜色为0,0,0,0(分别表示为红色，绿色，蓝色，透明度)。所以这个画布是透明的。

9、然后我们需要告诉它用哪个缓冲区和如何从缓冲区取出数据给到属性。
  // Bind the attribute/buffer set we want.
  gl.bindVertexArray(vao);

10、由于我们设置primitiveType的值为gl.TRIANGLES, 顶点着色器将会基于a_position设置的3对值画三角形。不管画布多大，这些值在裁剪空间坐标的范围是-1到1。

11、我希望你能看到WebGL实际上是很简单的API.简单的意思是，它仅仅运行两个函数(顶点着色器和片段着色器)来画三角形，线段和点。 然而3D绘制可以变得非常地复杂，这复杂性是由程序员来设计复杂着色器来实现的。 WebGL API仅仅是一个简单的栅格化工具(rasterizer)。

12、GPU 基本做了两部分事情： 第一部分是处理顶点(数据流)，变成裁剪空间节点；第二部分是基于第一部分的结果绘制像素。

13、左边是你提供的数据。点着色器是用GLSL写的函数。 每个顶点都用调用一次它。在这个函数里面， 做了一些数学运算和设置裁剪空间的顶点坐标 到一个特殊变量gl_position。GPU 获得了这些坐标值并在内部存起来。
  假设你在画一些三角形，每次 GPU 都会取出 3 个顶点来生成三角形。它指出三角形的 3 个点对应哪些像素， 然后这些像素值画出这个三角形，这个过程就叫“像素栅格化”。对于每个像素，都会调用片段着色器。 它有一个 vec4 类型的输出变量，它指示绘制像素的颜色是什么。

14、“varyings”就能够从点着色器传值到片段着色器。
  WebGL 将会连接在点着色器和片段着色器中拥有相同名称和类型的 varying 变量。

15、现在想一想，我们只计算了 3 个顶点。点着色器仅仅调用 3 次，也只计算 3 种颜色， 但是三角形却又许多种颜色。也就是我们叫它varying的原因。
  实际上，当 GPU 栅格化这个三角形的时候，它会基于这个三个顶点的颜色值做插值计算。 然后，WebGL 会基于这些插值来调用片段着色器。

16、缓冲区是将顶点和将每个顶点数据传给GPU的方法。 gl.createBuffer创建一个缓冲区。 gl.bindBuffer将该缓冲区设置为正在处理的缓冲区。 gl.bufferData将数据复制到当前缓冲区中。
  数据进入缓冲区后，我们需要告诉WebGL如何获取数据并将其提供给顶点着色器的属性。
  gl.vertexAttribPointer(
    location,
    numComponents,
    typeOfData,
    normalizeFlag,
    strideToNextPieceOfData,
    offsetIntoBuffer);
  上面这条命令告诉WebGL：从最后调用gl.bindBuffer绑定的缓冲区中获取数据； 每个顶点有多少个（1-4）分量；数据类型是什么（BYTE，FLOAT，INT，UNSIGNED_SHORT等）； 从一条数据到下一条数据需要跳过的字节数; 以及数据在缓冲区的偏移量。
  如果每种数据类型使用1个缓冲区，则步幅和偏移量都可以始终为0。 步幅0表示“使用与类型和大小匹配的步幅”。 偏移量为0表示从缓冲区的开头开始。 将它们设置为非0的值更为复杂，尽管在性能方面可能会有一些好处， 但是除非您试图将WebGL推向其绝对极限，否则不值得为此付出麻烦。

17、vertexAttribPointer中的normalizeFlag是什么？
  normalizeFlag适用于所有非浮点类型。 如果通过如果为false，则值将被解释为它们的类型。 BYTE从-128到127，UNSIGNED_BYTE从0到255，SHORT INTEGER从-32768到32767等...
  如果将normalize标志设置为true，则BYTE的值（-128至127）表示值-1.0至+ 1.0， UNSIGNED_BYTE（0至255）变为0.0至+1.0。标准化的SHORT INTEGER也从-1.0变为+1.0，它的分辨率比BYTE高。
  归一化数据的最常见用途是颜色。 大多数时候颜色仅从0.0到1.0。 如果红色，绿色，蓝色和Alpha分别使用一个完整的浮点数,每个顶点的每种颜色将使用16个字节。 如果您有复杂的几何体，将会可以增加很多字节。 相反，您可以将颜色转换为UNSIGNED_BYTEs (其中0代表0.0，255代表1.0)。 现在每种颜色只需要4个字节,每个顶点可节省75％的空间。
