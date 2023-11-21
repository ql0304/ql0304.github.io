# CoordinateSystem

任务：

- 生成并配置model，view，projection矩阵
- 开启深度测试

### model，view，projection矩阵

##### 生成model矩阵：

- 调用glm::rotate函数，glm::translate函数等。想怎么model就怎么model

```c++
	glm::mat4 model,view,projection;
    {
        model=glm::mat4(1.0f);
        model=glm::rotate(model,glm::radians(-55.0f),glm::vec3(1.f,0.f,0.f));//这里让物体绕(1.f,0.f,0.f)旋转-55度

        view=glm::mat4(1.f);
        glm::vec3 CameraPos=glm::vec3(0.f,-1.f,3.f);
        view=glm::translate(view,-CameraPos);
        view=glm::lookAt(CameraPos,glm::vec3(0.f,0.f,0.f),glm::vec3(0.f,1.0f,0.f));

        projection=glm::perspective(glm::radians(45.0f),(float)SCR_WIDTH/SCR_HEIGHT,0.1f,100.0f);

    }
```

- 可以在渲染循环中用(float)glfwGetTime()函数来获取时间，动态的修改model矩阵，移动场景中物体的位置。注意：每次循环要把model重新置为单位阵，免得旋转/移动的速度越来越快。

```c++
while (!glfwWindowShouldClose(window)){
        processInput();
        glClear(GL_COLOR_BUFFER_BIT|GL_DEPTH_BUFFER_BIT);

        shader.use();
        model=glm::mat4(1.f); //注意！
        model=glm::rotate(model,(float)glfwGetTime()*glm::radians(50.0f),glm::vec3(0.5f,1.f,0.f));
        shader.setMat4("model",model);
    
        glBindVertexArray(VAO);
        glDrawArrays(GL_TRIANGLES,0,36);

        glfwSwapBuffers(window);
        glfwPollEvents();
    }
```

##### 生成view矩阵：

- 先平移再旋转

- 平移的时候使用glm::translate()函数，参数为摄像机的位置。因为是要把摄像机移动到原点，所以这里是移动 -CameraPos。

- 旋转的时候需要使用lookAt()矩阵，设置相机位置、相机看向的点的位置以及up方向之后，该矩阵就能生成view坐标系，返回一个视图变换矩阵。

```c++
	glm::mat4 model,view,projection;
    {
        model=glm::mat4(1.0f);
        model=glm::rotate(model,glm::radians(-55.0f),glm::vec3(1.f,0.f,0.f));//这里让物体绕(1.f,0.f,0.f)旋转-55度

        view=glm::mat4(1.f);
        glm::vec3 CameraPos=glm::vec3(0.f,-1.f,3.f);
        view=glm::translate(view,-CameraPos);
        view=glm::lookAt(CameraPos,glm::vec3(0.f,0.f,0.f),glm::vec3(0.f,1.0f,0.f));

        projection=glm::perspective(glm::radians(45.0f),(float)SCR_WIDTH/SCR_HEIGHT,0.1f,100.0f);

    }
```

##### 生成透视投影矩阵projection：

这里分为透视投影和正交投影。透视投影矩阵使用glm::perspective()函数。参数：fovy，y方向视锥体张开的角度；aspect-ratio；znear；zfar

正交投影：使用glm::ortho()函数，参数为left，right，bottom，top，near，far6个float

对这些fovy与aspect-ratio进行实验，看看它们是如何影响透视平截头体的。

**fovy不会影响物体的形状，当fovy很小的时候有点像望远镜，物体离得很近，细节显示得很清楚。fovy很大的时候，感觉得到相机离物体很远，物体很小**

**aspect-ratio会改变物体的形状，当aspect-ratio变大，物体被左右压缩；当aspect-ratio变小，物体被上下压缩**

**注意：fovy一定要把角度转为弧度！！！**也就是要用glm::radians()函数。如果忘记转换直接填角度的话，滚轮往下滚1格，场景就会像下面这样很畸形

![mkdocs](images\perspectiveError.png)

弧度定义： 两条射线从圆心向圆周射出，形成一个夹角和夹角正对的一段弧。当这段弧长正好等于圆的半径时，两条射线的夹角大小为1弧度。弧度 = 弧长 / 半径

角度转弧度：**角度 \* (π / 180） = 弧度**

弧度转角度： **弧度 \* (180 / π） = 角度**

### 深度测试

OpenGL里的深度测试就很简单了。

- 开启深度测试。
- 在渲染循环中每次清空深度buffer。

```c++
glEnable(GL_DEPTH_TEST);  

while(1){   //渲染循环
	glClear(GL_COLOR_BUFFER_BIT|GL_DEPTH_BUFFER_BIT);
}
```
