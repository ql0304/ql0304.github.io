# Stencil Test

why：模板和模板测试 就是给对应的像素一个标记，根据这个标记，来决定这个像素是否渲染或者怎样渲染。可以用来

1. 可以用来对多重阴影绘制的去重
2. 可以用来决定是否绘制特定模型。常见的镂空渲染
3. 绘制物体的轮廓等等。。。

what：在片段着色器之后执行。模板测试和深度测试一样，有一个stencil buffer。buffer中的每一位都有**8**个比特(**1B**)，说明片段的模板值可以为0-255。我们可以设置当片段的模板值 等于(或不等于) 给定值的时候，通过模板测试。不通过模板测试的片段会被丢弃。怎么才能设置当前模板值呢？每一次渲染循环的我们都会用**glClear(GL_STENCIL_BUFFER_BIT)**;来把模板缓存中的模板值全部置为0。我们先在一个pass上，用**glStencilFunc**函数来给想要的片段设置想要的模板值。第二个pass，再进行模板测试。大致步骤如下：

- 启用模板缓冲的写入。
- 渲染物体，更新模板缓冲的内容。
- 禁用模板缓冲的写入。
- 渲染（其它）物体，这次根据模板缓冲的内容丢弃特定的片段。

how：以绘制物体轮廓为例。我们现在要给下图的箱子绘制轮廓。

![mkdocs](images\1.png)

1. 禁用模板写入，绘制地板。
2. 渲染箱子，让模板测试总是通过，并且给箱子覆盖的像素设置模板值为1。
3. 禁用深度测试
4. 再渲染一个放大一点的箱子，箱子颜色是给定的边框颜色，模板值为1则不通过模板测试(不绘制)，不为1则通过模板测试，把该片段绘制出来。

![mkocs](images\cube.jpg)

如图，绘制大一些的cube时，模板值为1的部分就不绘制。这样就能出现边框了

具体实现如下：

```c++
//开启深度测试
glEnable(GL_STENCIL_TEST);
glStencilFunc(GL_NOTEQUAL, 1, 0xFF);  //片段模板值和0xFF AND 之后(和0xFFand的结果其实就是自己)，若不为1，则不通过模板测试。
glStencilOp(GL_KEEP, GL_KEEP, GL_REPLACE);

while (!glfwWindowShouldClose(window)){
    //绘制floor的时候不希望对模板缓冲写入
    glStencilMask(0x00);
    //floor
    绘制floor。。。

    //1st pass，绘制箱子，绘制的时候设置箱子的模板值为1
    glStencilFunc(GL_ALWAYS, 1, 0xFF);//总是通过模板测试，模板值设为ref(第二个参数)
    glStencilMask(0xFF);     //启用模板写入
    //two cube
    绘制两个箱子。。。。。。

    //2nd pass，绘制两个大一点的箱子。
    glStencilFunc(GL_NOTEQUAL, 1, 0xFF);//模板值不为1的时候通过模板测试。
    glStencilMask(0x00);                //关闭模板写入
    glDisable(GL_DEPTH_TEST);           //关闭深度测试

    float scale = 1.1f;         //箱子放缩的大小
    // cubes
    model = glm::scale(model, glm::vec3(scale, scale, scale));
    shaderSingleColor.setMat4("model", model);
    绘制两个箱子。。。。。
    
    //重置模板写入和深度测试
    glStencilMask(0xFF);
    glStencilFunc(GL_ALWAYS, 0, 0xFF);
    glEnable(GL_DEPTH_TEST);

    glfwSwapBuffers(window);
    glfwPollEvents();
}
```
