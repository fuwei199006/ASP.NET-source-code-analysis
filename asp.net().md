### Asp.Net执行入口    

上面两篇文章说了http协议和IIS处理，这次说下当IIS把请求交给Asp.net后，后续的过程。首先要确定IIS是如何调用Asp.net的程序的。       
#### AppManagerAppDomain  

 1. 当IIS把请求交给asp.net时候，如果AppDomain还不存在则创建APPDomain，将AppDomain指派给与请求对应的应用程序，这通过AppManagerAppDomain类中的Create方法实现，代码如下：   
 
``` C#     
        public Object Create(String appId, String appPath) {
            try {
                //
                //  Fill app a Dictionary with 'binding rules' -- name value string pairs
                //  for app domain creation
                //

                // 


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
#### ISAPIRuntime    



