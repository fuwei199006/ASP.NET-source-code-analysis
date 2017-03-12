上次说到获得HttpApplication对象的创建，创建完成后调用InitInternal方法，下面详细过程。    

###InitInternal    

在这个方法任务比较多，也比较长，这里就不贴全码了，一个一个过程的去说：       

#### 初始化HttpModule      

##### httpModule使用实例    

   对于HttpModule，个人觉得应该首先看下是怎么使用的：   
   
  1. 新建一个项目，添加一个webform的窗体default.aspx       
  2. 添加一个MyModule.cs,继承自IHttpModule。     
  3. 在IHttpMoudule中有两个方法，在MyModule中必须要实现：         
  
  ``` C#       
    public void Init(HttpApplication context)
        {
            throw new System.NotImplementedException();
        }

        public void Dispose()
        {
            throw new System.NotImplementedException();
        }
  
  ```      
  4. 在Init方法中，有一个HttpApplication类型的对象context，这里可以对其中的响应的内容进行更改，修改如下：      
  ``` C#    
  
    public void Init(HttpApplication context)
    {
        context.EndRequest += Context_EndRequest;
    }

    private void Context_EndRequest(object sender, System.EventArgs e)
    {
        var context = (HttpApplication) sender;
        context.Response.Write("<h1>Hello MyModule</h1>");
    }
  
  ```        
  5. 添加web.config文件如下（在  <system.webServer><modules>节点下面）：    
  
  ``` xml    
   <add name="MyModule" type="Application.MyModule,Application"/>  
   
  ```    
  运行结果如下

##### asp.net中HttpModule的处理

1. HttpModule在Asp.net中有重要的作用，可以HttpApplication的事件进行订阅，也可以修改对应的响应的内容 

2. 对于HttpModule的初始化,asp.Net中会根据当前应用程序池的类型进行初始化，核心代码如下：      
``` C#    
   if (HttpRuntime.UseIntegratedPipeline) {
        try {
            context.HideRequestResponse = true;
            _hideRequestResponse = true;
            InitIntegratedModules();
        }
        finally {
            context.HideRequestResponse = false;
            _hideRequestResponse = false;
        }
    }
    else {
        InitModules();
    }
  }   
```   
3. 对于集成模式，获得所有Modules的方法是调用非托管的方法的进行获得，具体获得的代码如下:     

``` C#      

 [DllImport(_IIS_NATIVE_DLL)]
    internal static extern int MgdGetModuleCollection(
        IntPtr            pConfigSystem,
        IntPtr            appContext,
        out IntPtr        pModuleCollection,
        out int           count);


```    



#### Global内的方法调用     


#### 根据应用程序池的类型创建不同的_stepManager      


#### 执行BuildStep