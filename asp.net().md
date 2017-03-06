  

上面两篇文章说了http协议和IIS处理，这次说下当IIS把请求交给Asp.net后，后续的过程。首先要确定IIS是如何调用Asp.net的程序的。       
#### AppManagerAppDomain  

 1. 当IIS把请求交给asp.net时候，如果AppDomain还不存在则创建APPDomain，将AppDomain指派给与请求对应的应用程序，这通过AppManagerAppDomain类中的Create方法实现，代码如下：   
 
``` C#     
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
 2. 创建完成后，把请求交给ISAPIRuntime 中ProcessRequest方法     
 
#### ISAPIRuntime--asp.net入口    
 
 1.  首先看下ISAPIRuntime中的ProcessRequest方法签名   
 
 ``` C#     
  public int ProcessRequest(IntPtr ecb, int iWRType);
 
 ```        
 ProcessRequest有两个参数，一个是请求报文的ecb句柄，一个请求的线程的类型（此处尚有争议,对于源码的阅读没有实质的障碍），在运行的过程中，ecb首先被再次封装成托管资源的请求报文wr。 把封装好的代码传递给HttpRuntime类中的ProcessRequestNoDemand. 核心代码如下：   
 
 ``` C#   
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
 
 1. HttpRuntime收到传递过来的HttpWorkerRequest类的实例对象wr,通过调用当前类中的ProcessRequestNow方法，把参数传递给ProcessRequestInternal（ProcessRequestNow的调用了ProcessRequestInternal）。
 
 ``` C#  
 
 ```  
 
 2. 在ProcessRequestInternal方法的内部，实现对HttpContext类的对象实例的创建。

 
 



