# Blending

### 绘制草

现在想给之前的两个箱子场景加一些草，目标是这样的：

![mkdocs](images\2.png)

如果我们新建一个四边形的VAO，然后把草的纹理贴到四边形上，按之前的渲染步骤会得到下面这样的错误结果：

![mkdocs](images\3.jpg)

出现这种问题是因为OpenGL默认不知道如何处理alpha值，不知道什么时候该丢弃片段。所以要在片段着色器中手动指定。

```glsl
void main() {

    float alpha = texture(diffuse,texcoord).w;
    if(alpha < .5f) discard;   //0.5是阈值，0.5还是0.1随便指定 只要结果正确就可以

    FragColor = texture(diffuse,texcoord);

}
```

另一方面，加载有alpha值的纹理，要在纹理生成的时候，format用GL_RGBA

glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, data);

这样加载草就没有问题了。

### 绘制窗户

绘制草的时候，透明的片段我们直接丢弃了，但是这种方法不能让我们渲染半透明的图像。要想渲染有多个透明度级别的图像，我们需要启用混合(Blending)。具体如何启用请查阅[混合 - LearnOpenGL CN (learnopengl-cn.github.io)](https://learnopengl-cn.github.io/04 Advanced OpenGL/03 Blending/)。这里讲一下**最前面窗户的透明部分遮蔽了背后的窗户**这个问题。

![mkdocs](images\4.jpg)

窗户的前后顺序为1，2，3。可以看到1号窗户遮蔽了2号，但是2号却没遮住3号。这是为什么呢？这跟窗户的**绘制顺序**有关。

```c++
vector<glm::vec3> vegetation
{
    glm::vec3(-1.5f, 0.0f, -0.48f),    //红色
    glm::vec3( 1.5f, 0.0f, 0.51f),     //绿色
    glm::vec3( 0.0f, 0.0f, 0.7f),      //蓝色
    glm::vec3(-0.3f, 0.0f, -2.3f),     //白色
    glm::vec3 (0.5f, 0.0f, -0.6f)      //黑色
};

...
    
while(!glfwWindowShouldClose(window)){
    
    ...
        
    for(int i=0;i<vegetation.size();++i){
        model = glm::translate(glm::mat4(1.f),vegetation[i]);
        shader.setMat4("model",model);
        shader.setUniform3f("vegetationColor",vegetationColor[i]);
        //箱子的绘制顺序和vegetation里的顺序一致
        glDrawArrays(GL_TRIANGLES,0,6);
    }
}
```

为了方便发现问题，这里把窗户按绘制顺序涂成红、绿、蓝、白、黑色。白色的效果是不改变窗户本身的颜色。所以1，2，3号窗户的绘制顺序为：1，3，2。

1. 先绘制1号，1号深度最小，全部片段都会通过深度测试，此时场景中还没有2、3号窗户，所以1号的颜色和地板的颜色进行了blend；
2. 接下来绘制3号，3号与1号重叠的片段会在深度测试中被丢弃，因此1号遮蔽了3号；
3. 接下来绘制2号，2号与1号重叠的片段在深度测试中被丢弃，1号遮蔽2号；2号与3号重叠的片段则可以通过深度测试，所以3号的片段颜色和2号的片段颜色进行了blend，没有发生遮蔽。

结论：不能随意决定渲染透明物体的顺序，要手动的把窗户的顺序**从远到近排序**，再根据这个顺序绘制。对于不透明的物体，不需要blend，所以绘制顺序随意。当绘制一个有不透明和透明物体的场景的时候，大体的原则如下：

1. 先绘制所有不透明的物体。
2. 对所有透明的物体排序。
3. 从远到近按顺序绘制所有透明的物体。

![mkdocs](images\5.png)

vegetation按z坐标排序后就对了。

更高级的技术还有**次序无关透明度**(Order Independent Transparency, OIT)，先不展开。

# 4_4 Face_culling

用三角形三个顶点定义的顺序来表示这个三角形是正面还是背面。一般情况下，逆时针定义三角形时表示正面，顺时针定义三角形时表示背面。对于cube的这个例子：前面、右面、上面(能看到)按逆时针顺序定义；背面，左面，下面(不能看到)按顺时针顺序定义，再通知OpenGL就可以了。

主要有 3 条指令：

1. 启动，glEnable(GL_CULL_FACE);

2. 剔除正面或背面，glCullFace(GL_FRONT);

   - `GL_BACK`：只剔除背向面。

   - `GL_FRONT`：只剔除正向面。

   - `GL_FRONT_AND_BACK`：剔除正向面和背向面。

3. 正面被定义为顺时针还是逆时针，glFrontFace(GL_CCW); GL_CCW代表逆时针，GL_CW代表顺时针

把镜头伸到箱子里，可以感觉箱子好像凭空消失了，这是因为在箱子外面看不到的面，已经被剔除了。
