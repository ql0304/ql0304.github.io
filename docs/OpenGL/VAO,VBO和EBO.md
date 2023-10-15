# VAO,VBO和EBO

VBO：vertex buffer object。是一个纯数据缓冲区，存放顶点的信息数据，比如顶点坐标，顶点颜色。不同种类的信息可以放在一个VBO也可以分开放在多个VBO，全看VAO怎么解析。

VAO：vertex array object。VAO规定了VBO中数据的格式，比如当前顶点的起始地址和下一个顶点的起始地址之间的距离。此外，渲染管线在 顶点获取阶段 使用的object，VAO负责把顶点信息输入到shader中。

EBO：