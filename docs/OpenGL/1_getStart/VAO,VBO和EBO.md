# VAO,VBO和EBO

**VBO**：vertex buffer object。是一个纯数据缓冲区，存放顶点的信息数据，比如顶点坐标，顶点颜色。不同种类的信息可以放在一个VBO也可以分开放在多个，想怎么放就怎么放。

VBO的用法：

```c++
	unsigned int VBO;
    glGenBuffers(1,&VBO);
    glBindBuffer(GL_ARRAY_BUFFER,VBO);
    glBufferData(GL_ARRAY_BUFFER,sizeof(vertices),vertices,GL_STATIC_DRAW);
//注意，是bind到GL_ARRAY_BUFFER上
```

glBufferData是一个专门用来把用户定义的数据复制到当前绑定缓冲的函数。它的第一个参数是目标缓冲的类型：顶点缓冲对象当前绑定到GL_ARRAY_BUFFER目标上。第二个参数指定传输数据的大小(以字节为单位)；用一个简单的`sizeof`计算出顶点数据大小就行。第三个参数是我们希望发送的实际数据。

第四个参数指定了我们希望显卡如何管理给定的数据。它有三种形式：

- GL_STATIC_DRAW ：数据不会或几乎不会改变。

- GL_DYNAMIC_DRAW：数据会被改变很多。

- GL_STREAM_DRAW ：数据每次绘制时都会改变。

数据填充进去之后就要解析数据的哪一部分对应顶点着色器的哪个顶点属性了，这里使用**glVertexAttribPointer函数**

```c++
//glVertexAttribPointer函数重中之重！！！
void glVertexAttribPointer(GLuint index,GLint size,GLenum type,GLboolean normalized,GLsizei stride,const void * pointer);
//参数1：要配置的是哪个顶点属性，对应着色器中的layout(location = X)
//参数2：该顶点属性有几个分量
//3：数据类型
//4：是否自动进行单位化
//5：相邻两个属性之间的步长
//6：偏移量，表示位置数据在缓冲中起始位置的偏移量(Offset)
```

解析好数据之后，还要显式的启用它。因为顶点属性默认是禁用的

```c++
glEnableVertexAttribArray(X);
```

**VAO**：vertex array object。why：假设现在场景中有多个物体要绘制，有了vao之后，我们就可以在渲染循环前面只进行一次数据解析操作，交替绘制只需要在渲染循环中交替绑定vao就行。

**我理解的是VAO打包了给VBO填入并解析数据，给EBO填入并解析数据这一系列的动作。要换物体渲染的时候，只需换绑定的vao即可**

```c++
    float vertices[]={
            //position
        -0.5f, -0.5f, 0.0f,
        0.5f, -0.5f, 0.0f,
        0.0f,  0.5f, 0.0f
    };

    float colors[]={
            1.0f, 0.0f, 0.0f,
            0.0f, 1.0f, 0.0f,
            0.0f, 0.0f, 1.0f
    };

    float vertices2[]={
        -0.5f, 0.5f, 0.0f,
        0.5f, 0.5f, 0.0f,
        0.0f,  -0.5f, 0.0f
    };

	//第一个vao和它绑定的vbo
    unsigned int VAO,VBO;
    glGenVertexArrays(1,&VAO);
    glBindVertexArray(VAO);
    glGenBuffers(1,&VBO);
    glBindBuffer(GL_ARRAY_BUFFER,VBO);
    glBufferData(GL_ARRAY_BUFFER,sizeof(vertices),vertices,GL_STATIC_DRAW);
    glVertexAttribPointer(0,3,GL_FLOAT,GL_FALSE,3*sizeof(float),(void*)0);
    glEnableVertexAttribArray(0);
    unsigned int VBO1;
    glGenBuffers(1,&VBO1);
    glBindBuffer(GL_ARRAY_BUFFER,VBO1);
    glBufferData(GL_ARRAY_BUFFER,sizeof(colors),colors,GL_STATIC_DRAW);
    glVertexAttribPointer(1,3,GL_FLOAT,GL_FALSE,3*sizeof(float),(void*)0);
    glEnableVertexAttribArray(1);

	//第二个vao和它绑定的vbo
    unsigned int vao,vbo;
    glGenVertexArrays(1,&vao);
    glBindVertexArray(vao);
    glGenBuffers(1,&vbo);
    glBindBuffer(GL_ARRAY_BUFFER,vbo);
    glBufferData(GL_ARRAY_BUFFER,sizeof(vertices2),vertices2,GL_STATIC_DRAW);
    glVertexAttribPointer(0,3,GL_FLOAT,GL_FALSE,3*sizeof(float),(void*)0);
    glEnableVertexAttribArray(0);

    Shader shader("1_getStart/hello.vert","1_getStart/hello.frag");
    
    int i=0;
    while (!glfwWindowShouldClose(window)){
        processInput(window);
        glClear(GL_COLOR_BUFFER_BIT);

        //视口中交替显示VAO和vao中的数据绘制出来的图像
        if(i%2==0){
            glBindVertexArray(VAO);
        }else{
            glBindVertexArray(vao);
        }
        ++i;
        
        //vao的绑定必须在着色器的use之前
        shader.use();
        glDrawArrays(GL_TRIANGLES,0,3);

        glfwSwapBuffers(window);
        glfwPollEvents();
    }
```

**EBO：**element buffer object

why：当场景中有很多个三角形，三角形之间出现了顶点复用的情况。根据之前做法，我们只能在数组中重复存储这些顶点来表示哪三个顶点组成了一个三角形，显然这是非常浪费显存的。现在我们希望，只存储不同的顶点坐标，并**设定绘制这些顶点的顺序**。因此引出EBO

what：由名字可知，EBO也是一个数据缓冲区，它存储 OpenGL 用来决定要绘制哪些顶点的索引。FBO中的顶点用下标来编号，EBO中存储了每个三角形的顶点的编号。

how：和VBO类似，先绑定EBO然后用glBufferData把索引复制到缓冲里

```c++
	unsigned int EBO;
    glGenBuffers(1,&EBO);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER,EBO);    //注意：目标是GL_ELEMENT_ARRAY_BUFFER
    glBufferData(GL_ELEMENT_ARRAY_BUFFER,sizeof(index),index,GL_STATIC_DRAW);
```

**注意：在渲染循环里用glDrawElements函数，而不是glDrawArrays函数**

```c++
while (!glfwWindowShouldClose(window)){
        processInput(window);
        glClear(GL_COLOR_BUFFER_BIT);

        glBindVertexArray(VAO);
        shader.use();
//        glDrawArrays(GL_TRIANGLES,0,6);
        glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);

        glfwSwapBuffers(window);
        glfwPollEvents();
    }
```

```c++
void glDrawElements(GLenum mode,GLsizei count,GLenum type,const void * indices);
//参数2：要绘制多少个点；3：数据是什么类型，只能是GL_UNSIGNED_BYTE, GL_UNSIGNED_SHORT, or GL_UNSIGNED_INT.之一；4：先填0

```

