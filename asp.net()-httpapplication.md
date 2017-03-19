上次说到获得HttpApplication对象的创建，创建完成后调用InitInternal方法，下面详细过程。

### InitInternal方法

在这个方法任务比较多，也比较长，这里就不贴全码了，一个一个过程的去说：

#### 初始化HttpModule

##### httpModule使用实例

对于HttpModule，个人觉得应该首先看下是怎么使用的：

1. 新建一个项目，添加一个webform的窗体default.aspx，使用IIS添加到网站，应用程序池使用集成模式。
2. 添加一个MyModule.cs,继承自IHttpModule。
3. 在IHttpMoudule中有两个方法，在MyModule中必须要实现：

   ```C\#
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

   ```C\#
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
   运行结果：
![](/assets/HttpModule.png)
5. 添加web.config文件如下\(在 system.webServer下 modules节点下面\):

   ```xml
    <add name="MyModule" type="Application.MyModule,Application"/>
   ```

6. 在上面的例子中，使用的是集成模式，当改成经典模式的时候，module又不起作用了。对于经典模式的配置文件与集成模式不同。经典模式的配置文件如下：

   ```xml
    <httpModules>
    <add name="MyModule" type="IISIntegratedPipeline.MyModule,IISIntegratedPipeline"/>
    </httpModules>
   ```

7. 对于 module的使用，有了一个简单的认识，在asp.net中module是一个灵活的配置，可以对请求进行自定义的处理，对于Asp.net如何处理的，在下面详细解说。

##### asp.net中HttpModule的处理

1. 结合上面例子，HttpModule在Asp.net中有重要的作用，可以HttpApplication的事件进行订阅，也可以修改对应的响应的内容

2. 对于HttpModule的初始化,asp.Net中会根据当前应用程序池的类型进行初始化，核心代码如下：

   ```C\#
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
   ```

3. 对于Module的理解，需要根据应用程序池的模式来处理（经典和集成）。
4. 对于集成模式，获得所有Modules的方法是调用非托管的方法的进行获得，具体获得的代码如下:

5. InitIntegratedModules的方法

   ```C\#
     private void InitIntegratedModules()
     {
         _moduleCollection = BuildIntegratedModuleCollection(_moduleConfigInfo);
         InitModulesCommon();
     }
   ```

6. \_moduleConfigInfo 的来源  
   这个\_moduleConfigInfo的来源，还需要追到上篇 HttpApplication中三个方法的调用（EnsureAppStartCalled 第二个方法的调用）具体调用步骤如下：  
   EnsureAppStartCalled --&gt; FireApplicationOnStart --&gt; GetSpecialApplicationInstance --&gt; app.InitSpecial --&gt; RegisterEventSubscriptionsWithIIS--&gt; GetModuleCollection --&gt; GetConfigInfoForDynamicModules--&gt;UnsafeIISMethods.MgdGetModuleCollection 而最终的的调用来自于非托管代码

   \`\`\` C\#

   \[DllImport\(\_IIS\_NATIVE\_DLL\)\]  
     internal static extern int MgdGetModuleCollection\(  
     IntPtr pConfigSystem,  
     IntPtr appContext,  
     out IntPtr pModuleCollection,  
     out int count\);

    5. 对于经典模式获得Modules简单的多，直接获得是调用配置文件

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

1. 最终会调用 InitModulesCommon方法,循环调用Modules中的方法

   ```C\#
        private void InitModulesCommon()
        {
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

对于Global方法的调用，是调用HookupEventHandlersForApplicationAndModules\(handlers\);方法，这里的Handlers的收集和创建来源于上篇讲HttpAplication的三个方法调用的第一个方法。具体的看下上次的代码，这里不多叙述。对于方法的handlers的调用的核心代码如下,其实也是一个循环加上判断：

```C\#
    for (int i = 0; i < handlers.Length; i++) {
        MethodInfo appMethod = handlers[i];
        String appMethodName = appMethod.Name;
        int namePosIndex = appMethodName.IndexOf('_');
        String targetName = appMethodName.Substring(0, namePosIndex);

        ...
        ParameterInfo[] addMethodParams = addMethod.GetParameters();

        if (addMethodParams.Length != 1)
        continue;
        Delegate handlerDelegate = null;

        ParameterInfo[] appMethodParams = appMethod.GetParameters();

        ...
        try {
            addMethod.Invoke(target, new Object[1]{handlerDelegate});
        }
        catch {
            if (HttpRuntime.UseIntegratedPipeline) {
                throw;
            }
        }

    if (eventName != null) {
        if (_pipelineEventMasks.ContainsKey(eventName)) {
            if (!StringUtil.StringStartsWith(eventName, "Post")) {
            _appRequestNotifications |= _pipelineEventMasks[eventName];
            }
            else {
                _appPostNotifications |= _pipelineEventMasks[eventName];
            }
        }
    }
}
```

#### 根据应用程序池的类型创建不同的\_stepManager

这里很简单，直接看代码：

```C\#
// Construct the execution steps array
if (HttpRuntime.UseIntegratedPipeline) {
_stepManager = new PipelineStepManager(this);
}
else {
_stepManager = new ApplicationStepManager(this);
}
```

#### 执行BuildStep

BuildStep与ResumeStep是Asp.net的核心运行环节。同样，在经典模式与集成模式下原理和过程也有所不一样。

##### 集成模式

1. 下面先讨论集成模式下是如何进行的。

   \`\`\` C\#

        internal override void BuildSteps(WaitCallback stepCallback)
        {
            HttpApplication app = _application;

            IExecutionStep materializeStep = new MaterializeHandlerExecutionStep(app);

            app.AddEventMapping(
            HttpApplication.IMPLICIT_HANDLER,
            RequestNotification.MapRequestHandler,
            false, materializeStep);

            app.AddEventMapping(
            HttpApplication.IMPLICIT_HANDLER,
            RequestNotification.ExecuteRequestHandler,
            false, app.CreateImplicitAsyncPreloadExecutionStep());

            IExecutionStep handlerStep = new CallHandlerExecutionStep(app);

            app.AddEventMapping(
            HttpApplication.IMPLICIT_HANDLER,
            RequestNotification.ExecuteRequestHandler,
            false, handlerStep);

            IExecutionStep webSocketsStep = new TransitionToWebSocketsExecutionStep(app);

            app.AddEventMapping(
            HttpApplication.IMPLICIT_HANDLER,
            RequestNotification.EndRequest,
            true /* isPostNotification */, webSocketsStep);


            IExecutionStep filterStep = new CallFilterExecutionStep(app);
            app.AddEventMapping(
            HttpApplication.IMPLICIT_FILTER_MODULE,
            RequestNotification.UpdateRequestCache,
            false, filterStep);

            app.AddEventMapping(
            HttpApplication.IMPLICIT_FILTER_MODULE,
            RequestNotification.LogRequest,
            false, filterStep);

            _resumeStepsWaitCallback = stepCallback;
        }
    ```

1. 上面的代码是核心是AddEventMapping方法，把相关的步骤添加到一个PipelineModuleStepContainer.

   ```C\#
    private void AddEventMapping(string moduleName,RequestNotification requestNotification,bool isPostNotification, IExecutionStep step)
    {

        ThrowIfEventBindingDisallowed();
        if (!IsContainerInitalizationAllowed) {
            return;
        }
        PipelineModuleStepContainer container = GetModuleContainer(moduleName);
        if (container != null) {
            container.AddEvent(requestNotification, isPostNotification, step);
        }
    }
   ```

   ##### 经典模式

   1.看下代码：  
   \`\`\` C\#  
    internal override void BuildSteps\(WaitCallback stepCallback \) {  
        ArrayList steps = new ArrayList\(\);  
        HttpApplication app = \_application;

   ```
    bool urlMappingsEnabled = false;
    UrlMappingsSection urlMappings = RuntimeConfig.GetConfig().UrlMappings;
    urlMappingsEnabled = urlMappings.IsEnabled && ( urlMappings.UrlMappings.Count > 0 );

    steps.Add(new ValidateRequestExecutionStep(app));
    steps.Add(new ValidatePathExecutionStep(app));

    if (urlMappingsEnabled)
    steps.Add(new UrlMappingsExecutionStep(app)); // url mappings

    app.CreateEventExecutionSteps(HttpApplication.EventBeginRequest, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventAuthenticateRequest, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventDefaultAuthentication, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventPostAuthenticateRequest, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventAuthorizeRequest, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventPostAuthorizeRequest, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventResolveRequestCache, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventPostResolveRequestCache, steps);
    steps.Add(new MapHandlerExecutionStep(app)); // map handler
    app.CreateEventExecutionSteps(HttpApplication.EventPostMapRequestHandler, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventAcquireRequestState, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventPostAcquireRequestState, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventPreRequestHandlerExecute, steps);
    steps.Add(app.CreateImplicitAsyncPreloadExecutionStep()); // implict async preload step
    steps.Add(new CallHandlerExecutionStep(app)); // execute handler
    app.CreateEventExecutionSteps(HttpApplication.EventPostRequestHandlerExecute, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventReleaseRequestState, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventPostReleaseRequestState, steps);
    steps.Add(new CallFilterExecutionStep(app)); // filtering
    app.CreateEventExecutionSteps(HttpApplication.EventUpdateRequestCache, steps);
    app.CreateEventExecutionSteps(HttpApplication.EventPostUpdateRequestCache, steps);
    _endRequestStepIndex = steps.Count;
    app.CreateEventExecutionSteps(HttpApplication.EventEndRequest, steps);
    steps.Add(new NoopExecutionStep()); // the last is always there

    _execSteps = new IExecutionStep[steps.Count];
    steps.CopyTo(_execSteps);

    _resumeStepsWaitCallback = stepCallback;
   ```

   }

    对于上面的代码，可以看出都调用了 CreateEventExecutionSteps 方法，这个方法的详细如下 ：       
    ``` C#  
        private void CreateEventExecutionSteps(Object eventIndex, ArrayList steps) {
                // async
                AsyncAppEventHandler asyncHandler = AsyncEvents[eventIndex];
                if (asyncHandler != null) {
                    asyncHandler.CreateExecutionSteps(this, steps);
                }
                EventHandler handler = (EventHandler)Events[eventIndex];

                if (handler != null) {
                    Delegate[] handlers = handler.GetInvocationList();
                    for (int i = 0; i < handlers.Length; i++) {
                    steps.Add(new SyncEventExecutionStep(this, (EventHandler)handlers[i]));
                }
            }
        }

可以看出， CreateEventExecutionSteps是把注册的步骤都转换成了SyncEventExecutionStep，最终会被按顺序进行调用。

#### 执行BeginProcessRequest

HttpApplication在完成BuildSteps的时候，把生成的App经过层层返回到HttpRuntime,前面几篇文章提到，在HttpRuntime里面有对app的类型进行判断，如果是IHttpAsyncHandler直接调用BeginProcessRequest方法，具体的代码如下：

```C\#
        if (app is IHttpAsyncHandler) {
            IHttpAsyncHandler asyncHandler = (IHttpAsyncHandler)app;
            context.AsyncAppHandler = asyncHandler;
            asyncHandler.BeginProcessRequest(context, _handlerCompletionCallback, context);
        }
        else {
            // synchronous handler
            app.ProcessRequest(context);
            FinishRequest(context.WorkerRequest, context, null);
        }
```

BeginProcessRequest 方法：

```C\#
IAsyncResult IHttpAsyncHandler.BeginProcessRequest(HttpContext context, AsyncCallback cb, Object extraData) {
        HttpAsyncResult result;


        _context = context;
        _context.ApplicationInstance = this;

        _stepManager.InitRequest();

        _context.Root();

        result = new HttpAsyncResult(cb, extraData);

        AsyncResult = result;

        if (_context.TraceIsEnabled)
        HttpRuntime.Profile.StartRequest(_context);

        ResumeSteps(null);
        return result;
    }
```

其中最核心的方法是ResumeSteps方法，具体如下：

```C\#
internal override void ResumeSteps(Exception error)
{

    for (; ; ) {

        // ...

        IExecutionStep step = _application.CurrentModuleContainer.GetNextEvent(context.CurrentNotification, context.IsPostNotification,context.CurrentModuleEventIndex);

        context.SyncContext.Enable();

        stepCompletedSynchronously = false;

        //*******
        error = _application.ExecuteStep(step, ref stepCompletedSynchronously);
        //*********
        ...

        if (!stepCompletedSynchronously) {
            _application.AcquireNotifcationContextLock(ref locked);
            context.NotificationContext.PendingAsyncCompletion = true;
            break;
        }
        else {
            context.Response.SyncStatusIntegrated();
        }
    }
}
```

BeginProcessRequest 方法：

```C\#
IAsyncResult IHttpAsyncHandler.BeginProcessRequest(HttpContext context, AsyncCallback cb, Object extraData) {
    HttpAsyncResult result;


    _context = context;
    _context.ApplicationInstance = this;

    _stepManager.InitRequest();

    _context.Root();

    result = new HttpAsyncResult(cb, extraData);

    AsyncResult = result;

    if (_context.TraceIsEnabled)
    HttpRuntime.Profile.StartRequest(_context);

    ResumeSteps(null);
    return result;
}
```

其中最核心的方法是ResumeSteps方法，具体如下：

```C\#
    internal override void ResumeSteps(Exception error)
    {
        ...
        for (; ; ) {

        // ...

        IExecutionStep step = _application.CurrentModuleContainer.GetNextEvent(context.CurrentNotification, context.IsPostNotification,context.CurrentModuleEventIndex);

        context.SyncContext.Enable();

        stepCompletedSynchronously = false;

        //*******
        error = _application.ExecuteStep(step, ref stepCompletedSynchronously);
        //*********
        ...

        if (!stepCompletedSynchronously) {
            _application.AcquireNotifcationContextLock(ref locked);
            context.NotificationContext.PendingAsyncCompletion = true;
            break;
        }
        else {
            context.Response.SyncStatusIntegrated();
            }
        }
    }
```   
根据今天的分析，可以把上次的图更加完善下：    
![](/assets/Asp.net2.png)

 

