# Shader

**给着色器传uniform参数**

- 用函数**glGetUniformLocation**，根据参数名称获取位置值
- 用**glUniformXX**函数，根据获取到的location传入数据
- 更新一个uniform之前你**必须**先使用程序（调用glUseProgram)

```c++
//例子
	void setUniform3f(const char* name,float x,float y,float z){
        int location = glGetUniformLocation(ID,name);
    	if(location==-1){                       //没找到location
            cout<<"DO NOT FIND LOCATION"<<endl;
            return;
        }
        glUseProgram(ID);
        glUniform3f(location,x,y,z);
    }
```

