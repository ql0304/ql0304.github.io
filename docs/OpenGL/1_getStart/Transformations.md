# Transformations

变换：指缩放，位移，旋转，切变，仿射变换等等。需要用矩阵和向量的概念和知识，这里就不讲矩阵和向量了。

第三方库：GLM，下载解压之后把下图文件夹复制到include文件夹下就能使用了

![mkdocs](images\glmLoad.png)

大多数功能都在下面这3个头文件中找到

```c++
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
#include <glm/gtc/type_ptr.hpp>
```

## 在顶点着色器中修改顶点的位置，使其旋转移动

在顶点着色器中，每个顶点坐标都需要乘一个变换矩阵，因此要用uniform，把一个4*4的矩阵传进去

```
uniform mat4 trans;
```

shader类中要添加传参进去的函数

```c++
	void setMat4(const char* name,glm::mat4 value){
        int location = glGetUniformLocation(ID,name);
        if(location==-1){
            cout<<"DO NOT FIND LOCATION"<<endl;
            return;
        }
        glUniformMatrix4fv(location,1,GL_FALSE,glm::value_ptr(value));
    }
void glUniformMatrix4fv(GLint location,GLsizei count,GLboolean transpose,const GLfloat *value);
//count:要传入的矩阵的数量，大于1的时候，value要为矩阵数组指针
//transpose:是否要将矩阵转置
```

生成一个变换矩阵

```c++
	//变换矩阵
    glm::mat4 trans = glm::mat4(1.0f);   //一定要给trans初始化为单位阵！不然默认是0矩阵，做任何操作都是0！
    trans = glm::rotate(trans,glm::radians(90.0f),glm::vec3(0.0,0.0,1.0));
    trans = glm::scale(trans,glm::vec3(0.5,0.5,0.5));
	//传入shader
	shader.use();
    shader.setMat4("transform",trans);
```

写在渲染循环中，可以加一些随时间变化的量，使得物体移动起来
