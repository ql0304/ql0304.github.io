# ViewerBase::frame()

![mkdocs](images\3.png)

1. 若是第一帧，就初始化viewer。
2. 执行advance
3. event遍历
4. 更新遍历；
5. 渲染遍历

## 初始化：viewerInit()

viewerInit()函数只调用了View::init()函数。该函数出现了两个重要的类成员变量：**\_eventQueue** 和  **_cameraManipulator**。

\_eventQueue 创建出了一个帧类型的事件(**FRAME事件**)，即每一帧都会触发的事件。

```c++
osg::ref_ptr<osgGA::GUIEventAdapter> initEvent = _eventQueue->createEvent();
initEvent->setEventType(osgGA::GUIEventAdapter::FRAME);
```

osgGA::GUIEventAdapter代表事件，表达了各类型鼠标键盘等事件。用户可以通过继承**osgGA::GUIEventHandler类**并重写handle方法，来获取事件并处理事件。

\_cameraManipulator是view所用的漫游器的实例，通常用**setCameraManipulator**来设置这个变量的内容。漫游器在osgGA库中实现，这个库主要是用来处理用户与三维场景的交互（包括鼠标、键盘、手势、操纵杆等），osg中实现了很多类漫游器，这些漫游器都继承自osgGA::CameraManpualtor。

![mkdocs](images\1.jpg)

osgGA::CameraManpualtor继承自osgGA::GUIEventHandler，GUIEventHandler是用来处理事件的类，所以**漫游器实际上就是一个交互操作修改场景中节点位置和姿态的类，只不过它修改的是最顶层的相机节点**。说白了，漫游器修改的是OpenGL中的view矩阵。

View::init()处理完 \_eventQueue 之后，要把FRAME事件和viewer对象本身传递给\_cameraManipulator的init函数

```c++
if (_cameraManipulator.valid()){
    _cameraManipulator->init(*initEvent, *this);
}
```

## **osgViewer::Viewer::realize()**

由第一张图可知该函数只在第一帧的时候执行。其功能是完成窗口和场景的**“设置”**工作。realize首先完成的工作就是获取图形上下文**osg::GraphicsContext**。Contexts是保存了指针osg::ref_ptr\<osg::GraphicsContext>的数组。getContexts获取了主相机和所有从相机的图形上下文，排序之后set到contexts里。

```c++
void Viewer::realize()
{
    //OSG_INFO<<"Viewer::realize()"<<std::endl;

    Contexts contexts;
    getContexts(contexts);

    //如果getContexts没获得任何图形上下文，那么就创建缺省GraphicsContext
    if (contexts.empty())
    {
        OSG_INFO<<"Viewer::realize() - No valid contexts found, setting up view across all screens."<<std::endl;

        // no windows are already set up so set up a default view

        std::string value;
        if (osg::getEnvVar("OSG_CONFIG_FILE", value)) //1.从环境变量OSG_CONFIG_FILE读取文件路径
        {
            //1.调用 osgDB::readObjectFile 函数读取文件，用插件解析
            //2.若成功，调用take函数设置当前的viewer
            readConfiguration(value);
        }
        else
        {
            int screenNum = -1;
            osg::getEnvVar("OSG_SCREEN", screenNum);

            int x = -1, y = -1, width = -1, height = -1;
            osg::getEnvVar("OSG_WINDOW", x, y, width, height);

            if (osg::getEnvVar("OSG_BORDERLESS_WINDOW", x, y, width, height))//2.从环境变量OSG_BORDERLESS_WINDOW读取
            {
                osg::ref_ptr<osgViewer::SingleWindow> sw = new osgViewer::SingleWindow(x, y, width, height, screenNum);
                sw->setWindowDecoration(false);
                apply(sw.get());
            }
            else if (width>0 && height>0)//3.从OSG_WINDOW读取
            {
                //如果设置了OSG_WINDOW的同时也设置了OSG_SCREEN，调用setUpViewInWindow函数
                if (screenNum>=0) setUpViewInWindow(x, y, width, height, screenNum);
                else setUpViewInWindow(x,y,width,height); //没设置OSG_SCREEN，则调用setUpViewInWindow函数
            }
            else if (screenNum>=0) //4.从OSG_SCREEN读取，调用setUpViewOnSingleScreen函数
            {
                setUpViewOnSingleScreen(screenNum);
            }
            else  //5.上述环境变量都没设置
            {
                setUpViewAcrossAllScreens();
            }
        }
        getContexts(contexts);
    }

    //如果创建缺省GraphicsContext失败，退出程序
    if (contexts.empty())
    {
        OSG_NOTICE<<"Viewer::realize() - failed to set up any windows"<<std::endl;
        _done = true;
        return;
    }
    ......
}
```

值得注意的是setUpViewInWindow()函数，第一步是新建一个显示设备特性实例

```c++
osg::ref_ptr<osg::GraphicsContext::Traits> traits = new osg::GraphicsContext::Traits;
```

个traits里面的成员变量设置想要的值。

2.创建GraphicsContext，把traits作为参数传进去。把这个新建的GraphicsContext传给camera

```c++
osg::ref_ptr<osg::GraphicsContext> gc = osg::GraphicsContext::createGraphicsContext(traits.get());
_camera->setGraphicsContext(gc.get());
```

注意不要用new来创建osg::GraphicsContext，因为createGraphicsContext函数还做了：1.获取窗口系统api接口2.若用户没设置屏幕数，缺省为0；3.返回实例

3.调用 osgGA::GUIEventAdapter::setWindowRectangle 记录新建立的窗口设备的大小，因而这个设备上产生的键盘和鼠标事件可以以此为依据。

4.设置主摄像机\_camera 的透视投影参数，并设置新的 Viewport 视口。

5.执行 osg::Camera::setDrawBuffer 和执行 osg::Camera:: setReadBuffer 函数。

![mkdocs](images\4.png)

```c++
void Viewer::realize()
{
    
    ......
        
	// get the display settings that will be active for this viewer
    osg::DisplaySettings* ds = _displaySettings.valid() ? _displaySettings.get() : osg::DisplaySettings::instance().get();
    osg::GraphicsContext::WindowingSystemInterface* wsi = osg::GraphicsContext::getWindowingSystemInterface();

    // pass on the display settings to the WindowSystemInterface.
    if (wsi && wsi->getDisplaySettings()==0) wsi->setDisplaySettings(ds);

    unsigned int maxTexturePoolSize = ds->getMaxTexturePoolSize();
    unsigned int maxBufferObjectPoolSize = ds->getMaxBufferObjectPoolSize();

    for(Contexts::iterator citr = contexts.begin();
        citr != contexts.end();
        ++citr)
    {
        osg::GraphicsContext* gc = *citr;

        if (ds->getSyncSwapBuffers()) gc->setSwapCallback(new osg::SyncSwapBuffersCallback);

        // set the pool sizes, 0 the default will result in no GL object pools.
        gc->getState()->setMaxTexturePoolSize(maxTexturePoolSize);
        gc->getState()->setMaxBufferObjectPoolSize(maxBufferObjectPoolSize);

        gc->realize();

        if (_realizeOperation.valid() && gc->valid())
        {
            gc->makeCurrent();

            (*_realizeOperation)(gc);

            gc->releaseContext();
        }
    }
```

realize函数的剩下部分先跳过吧，看不懂了。。。

## 真正的一帧组成部分

**advance，eventTraversal，updateTraversal 和 renderingTraversals 函数**

advance主要是用来计算帧率的；

osg::ref_ptr\<T>采用引用计数。osg的智能指针不能用delete方法来删除，因为OSG采用多线程更新/渲染的方式，所以存在**“上次的渲染工作与下次的更新工作交叠”**这种情形，如果在更新工作中就把节点删除，而上次的节点渲染工作正要送到OpenGL，这时就会出错。解决办法是在渲染后台也使用 ref_ptr来引用（ref）图形节点，然后在渲染结束取消引用（unref）。但是每个节点的渲染都要多执行一次 ref/unref 的话，效率会下降很多，因此新版OSG提出了 **DeleteHandler** 的概念，也就是“垃圾收集”，把那些引用计数已经为零的对象统一收集起来，确保它们不会再被渲染线程用到之后，再在适当的地方予以释放。DeleteHandler 有一个重要的参数_numFramesToRetainObjects，它的意义是，垃圾对象被收集之后，再经过多少帧（默认设置是 2），方予以释放。

### viewer、camera和scene的关系

![mkdocs](images\2.png)

eventTraversal的主要任务：它必须在每一帧的仿真过程中，取出已经发生的所有事件，摒弃那些对场景不会有助益的（例如，在视口以外发生的鼠标移动事件和胡乱点击），依次交付给各个事件处理器，最后清空现有的事件队列，等待下一帧的到来。事件遍历，除了可以执行用户设置的事件回调（EventCallback）之外，更重要的工作是为所有的用户交互和系统事件提供一个响应的机制。