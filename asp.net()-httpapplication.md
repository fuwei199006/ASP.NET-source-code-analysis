上次说到获得HttpApplication对象的创建，创建完成后调用InitInternal方法，下面详细过程。    

###InitInternal    

在这个方法任务比较多，也比较长，这里就不贴全码了，一个一个过程的去说：       

#### 初始化HttpModule      

##### httpModule使用实例    

   对于HttpModule，个人觉得应该首先看下是怎么使用的：   
   
  1. 新建一个项目，添加一个webform的窗体default.aspx，使用IIS添加到网站，应用程序池使用集成模式。      
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
  运行结果如下：   
  ![](/assets/HttpModule.png)     
  
  6.在上面的例子中，使用的是集成模式，当改成经典模式的时候，module又不起作用了。对于经典模式的配置文件与集成模式不同。经典模式的配置文件如下：    
  
  ``` xml    
<httpModules>
        <add name="MyModule" type="IISIntegratedPipeline.MyModule,IISIntegratedPipeline"/>
</httpModules>
  
  ```      
  
  7.对于 module的使用，有了一个简单的认识，在asp.net中module是一个灵活的配置，可以对请求进行自定义的处理，对于Asp.net如何处理的，在下面详细解说。
  

##### asp.net中HttpModule的处理

1. 结合上面例子，HttpModule在Asp.net中有重要的作用，可以HttpApplication的事件进行订阅，也可以修改对应的响应的内容 

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
3. 对于Module的理解，需要根据应用程序池的模式来处理（经典和集成）。
4. 对于集成模式，获得所有Modules的方法是调用非托管的方法的进行获得，具体获得的代码如下:     

    - InitIntegratedModules的方法       
    ``` C#    
         private void InitIntegratedModules() 
         {
            _moduleCollection = BuildIntegratedModuleCollection(_moduleConfigInfo);
            InitModulesCommon();
         }   
    ```       
    - _moduleConfigInfo 的来源    
     这个_moduleConfigInfo的来源，还需要追到上篇 HttpApplication中三个方法的调用（EnsureAppStartCalled 第二个方法的调用）具体调用步骤如下：     
     
     EnsureAppStartCalled --> FireApplicationOnStart --> GetSpecialApplicationInstance -->  app.InitSpecial --> RegisterEventSubscriptionsWithIIS--> GetModuleCollection --> GetConfigInfoForDynamicModules-->UnsafeIISMethods.MgdGetModuleCollection  而最终的的调用来自于非托管代码 
    

``` C#      

 [DllImport(_IIS_NATIVE_DLL)]
    internal static extern int MgdGetModuleCollection(
        IntPtr            pConfigSystem,
        IntPtr            appContext,
        out IntPtr        pModuleCollection,
        out int           count);


```     
5.对于经典模式获得Modules简单的多，直接获得是调用配置文件    

 ``` C#    
  private void InitModules()
  {
    HttpModulesSection pconfig = RuntimeConfig.GetAppConfig().HttpModules;

    // get the static list, then add the dynamic members
    HttpModuleCollection moduleCollection = pconfig.CreateModules();
    HttpModuleCollection dynamicModules = CreateDynamicModules();

    moduleCollection.AppendCollection(dynamicModules);
    _moduleCollection = moduleCollection; // don't assign until all ops have succeeded

    InitModulesCommon();
   }
 
 ```     
 
 6. 最终会调用 InitModulesCommon方法,循环调用Modules中的方法    
 
 ``` C#   
       private void InitModulesCommon() {
            int n = _moduleCollection.Count;

            for (int i = 0; i < n; i++) {
                // remember the module being inited for event subscriptions
                // we'll later use this for routing
                _currentModuleCollectionKey = _moduleCollection.GetKey(i);
                _moduleCollection[i].Init(this);
            }

            _currentModuleCollectionKey = null;
            InitAppLevelCulture();
        }
 
 ``` 



#### Global内的方法调用     


#### 根据应用程序池的类型创建不同的_stepManager      


#### 执行BuildStep