上面两篇文章说了http协议和IIS处理，这次说下当IIS把请求交给Asp.net后，后续的过程。首先要确定IIS是如何调用Asp.net的程序的。

#### AppManagerAppDomain

1. 当IIS把请求交给asp.net时候，如果AppDomain还不存在则创建APPDomain，将AppDomain指派给与请求对应的应用程序，这通过AppManagerAppDomain类中的Create方法实现，代码如下：   

```C\#
        public Object Create(String appId, String appPath) {
            try {

                if (appPath[0] == '.') {
                    System.IO.FileInfo file = new System.IO.FileInfo(appPath);
                    appPath = file.FullName;
                }

                if (!StringUtil.StringEndsWith(appPath, '\\')) {
                    appPath = appPath + "\\";
                }

                // Create new app domain via App Manager
#if FEATURE_PAL // FEATURE_PAL does not enable IIS-based hosting features
                throw new NotImplementedException("ROTORTODO");
#else // FEATURE_PAL 

                ISAPIApplicationHost appHost = new ISAPIApplicationHost(appId, appPath, false /*validatePhysicalPath*/);

                //创建环境，包括编译环境
                ISAPIRuntime isapiRuntime = (ISAPIRuntime)_appManager.CreateObjectInternal(appId, typeof(ISAPIRuntime), appHost, 
                        false /*failIfExists*/, null /*hostingParameters*/);


                isapiRuntime.StartProcessing();

                return new ObjectHandle(isapiRuntime);
#endif // FEATURE_PAL 
            }
            catch (Exception e) {
                Debug.Trace("internal", "AppDomainFactory::Create failed with " + e.GetType().FullName + ": " + e.Message + "\r\n" + e.StackTrace);
                throw;
            }
        }
```

1. 创建完成后，把请求交给ISAPIRuntime 中ProcessRequest方法     

#### ISAPIRuntime--asp.net入口

1. 首先看下ISAPIRuntime中的ProcessRequest方法签名

   ```C\#
   public int ProcessRequest(IntPtr ecb, int iWRType);
   ```

   ProcessRequest有两个参数，一个是请求报文的ecb句柄，一个请求的线程的类型（此处尚有争议,对于源码的阅读没有实质的障碍），在运行的过程中，ecb首先被再次封装成托管资源的请求报文wr。 把封装好的代码传递给HttpRuntime类中的ProcessRequestNoDemand. 核心代码如下：

   ```C\#
   bool useOOP = (iWRType == WORKER_REQUEST_TYPE_OOP);
   wr = ISAPIWorkerRequest.CreateWorkerRequest(ecb, useOOP);
   wr.Initialize();

   // check if app path matches (need to restart app domain?)                
   String wrPath = wr.GetAppPathTranslated();
   String adPath = HttpRuntime.AppDomainAppPathInternal;

   if (adPath == null ||
      StringUtil.EqualsIgnoreCase(wrPath, adPath))
   {

      HttpRuntime.ProcessRequestNoDemand(wr);
      return 0;
   }
   else
   {
      // need to restart app domain
      HttpRuntime.ShutdownAppDomain(ApplicationShutdownReason.PhysicalApplicationPathChanged,
                                    SR.GetString(SR.Hosting_Phys_Path_Changed,
                                                                     adPath,
                                                                     wrPath));
      return 1;
   }
   ```

   #### HttpRuntime

2. HttpRuntime收到传递过来的HttpWorkerRequest类的实例对象wr,通过调用当前类中的ProcessRequestNow方法，把参数传递给ProcessRequestInternal（ProcessRequestNow的调用了ProcessRequestInternal）。

   ```C\#
       internal static void ProcessRequestNoDemand(HttpWorkerRequest wr) {
           RequestQueue rq = _theRuntime._requestQueue;

           wr.UpdateInitialCounters();

           if (rq != null)  // could be null before first request
               wr = rq.GetRequestToExecute(wr);

           if (wr != null) {
               CalculateWaitTimeAndUpdatePerfCounter(wr);
               wr.ResetStartTime();
               ProcessRequestNow(wr);
           }
       }

       internal static void ProcessRequestNow(HttpWorkerRequest wr) {
           _theRuntime.ProcessRequestInternal(wr);
       }
   ```

3. 在ProcessRequestInternal中，创建了HttpContext和HttpApplication对象实例，核心代码如下    

    ``` C#   
 
        private void ProcessRequestInternal(HttpWorkerRequest wr) {
         
         ...
        
        // Construct the Context on HttpWorkerRequest, hook everything together
        HttpContext context;
        
        try {
            context = new HttpContext(wr, false /* initResponseWriter */);
        } 
        catch {
            ...
        }
        
        ...
        
        try {
              ...
            // Get application instance
            IHttpHandler app = HttpApplicationFactory.GetApplicationInstance(context);
        
            if (app == null)
                throw new HttpException(SR.GetString(SR.Unable_create_app_object));
        
             ... 
        
            if (app is IHttpAsyncHandler) {
                // asynchronous handler
                IHttpAsyncHandler asyncHandler = (IHttpAsyncHandler)app;
                context.AsyncAppHandler = asyncHandler;
                asyncHandler.BeginProcessRequest(context, _handlerCompletionCallback, context);
            }
            else {
                // synchronous handler
                app.ProcessRequest(context);
                FinishRequest(context.WorkerRequest, context, null);
            }
        }
        catch (Exception e) {
            ...
        }
      }
    ```      
4. 在ProcessRequestInternal方法的内部，实现对HttpContext类和HttpApplicationFactory的对象实例的创建，核心代码: 根据上面代码，当获得HttApplication对象后，判断是否是IHttpAsyncHandler类型，如果是则调用BeginProcessRequest方法，此处的if条件是一直成立的，因为HttpApplication实现了IHttpAsyncHandler接口,而ProcessRequest方法的实现也仅仅是抛出了一个异常，笔者觉得此处应该是微软留了一个扩展的地方。    

    ``` C#  
      public class HttpApplication : IComponent, 
        IHttpAsyncHandler, IRequestCompletedNotifier, ISyncContext {
         ...
    
        }
    
    ```   
    
    ``` C#  
       void IHttpHandler.ProcessRequest(HttpContext context) {
                throw new HttpException(SR.GetString(SR.Sync_not_supported));
            }
    ```     


#### HttpContext对象

这个对象是一个请求响应的结合体，里面包含了HttpRequest和HttpResponse对象，在构造HttpContext对象时，同时也对HttpRequest和HttpResponse也进行了初始化，代码如下：   

    ``` C#     
        internal HttpContext(HttpWorkerRequest wr, bool initResponseWriter) {
            _wr = wr;
            Init(new HttpRequest(wr, this), new HttpResponse(wr, this));

            if (initResponseWriter)
                _response.InitResponseWriter();

            PerfCounters.IncrementCounter(AppPerfCounter.REQUESTS_EXECUTING);
        }
    
    ```    
    
#### 创建HttpApplication      

1. 通过HttpApplicationFactory中的静态方法GetApplicationInstance来获得实例对象（常用的工厂模式），在创建对象的时候调用了 _theApplicationFactory.GetNormalApplicationInstance(context);方法(其中context形参是上文创建的HttpContext）来执行实例化操作，核心代码如下：     

 ``` C#       
      
        internal static IHttpHandler GetApplicationInstance(HttpContext context) {
            if (_customApplication != null)
                return _customApplication;

            // Check to see if it's a debug auto-attach request
            if (context.Request.IsDebuggingRequest)
                return new HttpDebugHandler();
            
            _theApplicationFactory.EnsureInited();

            _theApplicationFactory.EnsureAppStartCalled(context);

            return _theApplicationFactory.GetNormalApplicationInstance(context);
        }
        
         private void Init() {
            if (_customApplication != null)
                return;

            try {
                try {
                    _appFilename = GetApplicationFile();

                    //编译
                    CompileApplication();
                }
                finally {
                    // Always set up global.asax file change notification, even if compilation
                    // failed.  This way, if the problem is fixed, the appdomain will be restarted.
                    SetupChangesMonitor();
                }
            }
            catch { // Protect against exception filters
                throw;
            }
        }

 ```          
 
 这个方法里有三个方法的调用，分别是：
 
 - _theApplicationFactory.EnsureInited();    
 
     主要是对Global.asxc文件进行编译和处理，核心代码如下：   
     
     ``` C#   
        private void EnsureInited() {
            if (!_inited) {
                lock (this) {
                    if (!_inited) {
                        Init();
                        _inited = true;
                    }
                }
            }
        }  
        
      private void Init() {
            if (_customApplication != null)
                return;

            try {
                try {
                    _appFilename = GetApplicationFile();

                    //编译
                    CompileApplication();
                }
                finally {
                    // Always set up global.asax file change notification, even if compilation
                    // failed.  This way, if the problem is fixed, the appdomain will be restarted.
                    SetupChangesMonitor();
                }
            }
            catch { // Protect against exception filters
                throw;
            }
        }
     
     ```      
     
  
 
 - _theApplicationFactory.EnsureAppStartCalled(context);    
    创建特定的HttpApplication实例，触发ApplicationOnStart事件，执行ASP.global_asax中的Application_Start(object sender, EventArgs e)方法。这里创建的HttpApplication实例在处理完事件后，就被回收。 具体实现：   
    
    ``` C#  
    
     private void EnsureAppStartCalled(HttpContext context) {
            if (!_appOnStartCalled) {
                lock (this) {
                    if (!_appOnStartCalled) {
                        using (new DisposableHttpContextWrapper(context)) {
                            // impersonation could be required (UNC share or app credentials)

                            WebBaseEvent.RaiseSystemEvent(this, WebEventCodes.ApplicationStart);

                            // fire outside of impersonation as HttpApplication logic takes
                            // care of impersonation by itself
                            FireApplicationOnStart(context);
                        }

                        _appOnStartCalled = true;
                    }
                }
            }
        }  
    ```
 - _theApplicationFactory.GetNormalApplicationInstance(context);     
 
   对于这个方法的理解，需要看下对应的代码：   
   
 ``` C#  
 
    private HttpApplication GetNormalApplicationInstance(HttpContext context) {
            HttpApplication app = null;

            lock (_freeList) {
                if (_numFreeAppInstances > 0) {
                    app = (HttpApplication)_freeList.Pop();
                    _numFreeAppInstances--;

                    if (_numFreeAppInstances < _minFreeAppInstances) {
                        _minFreeAppInstances = _numFreeAppInstances;
                    }
                }
            }

            if (app == null) {
                // If ran out of instances, create a new one
                app = (HttpApplication)HttpRuntime.CreateNonPublicInstance(_theApplicationType);

                using (new ApplicationImpersonationContext()) {
                   // 调用BuildSteps和获得所有的HttpModule
                    app.InitInternal(context, _state, _eventHandlerMethods);
                }
            }

            if (AppSettings.UseTaskFriendlySynchronizationContext) {
                // When this HttpApplication instance is no longer in use, recycle it.
                app.ApplicationInstanceConsumersCounter = new CountdownTask(1); // representing required call to HttpApplication.ReleaseAppInstance
                app.ApplicationInstanceConsumersCounter.Task.ContinueWith((_, o) => RecycleApplicationInstance((HttpApplication)o), app, TaskContinuationOptions.ExecuteSynchronously);
            }
            return app;
        }
 
 ```     
 

 

