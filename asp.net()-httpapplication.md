上次说到获得HttpApplication对象的创建，创建完成后调用InitInternal方法，下面详细过程。    

###InitInternal    

在这个方法任务比较多，也比较长，这里就不贴全码了，一个一个过程的去说：       

#### 初始化HttpModule      

##### httpModule使用实例  

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
  }   if (HttpRuntime.UseIntegratedPipeline) {
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



#### Global内的方法调用     


#### 根据应用程序池的类型创建不同的_stepManager      


#### 执行BuildStep