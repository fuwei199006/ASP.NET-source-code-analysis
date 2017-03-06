### Asp.Net执行入口    

上面两篇文章说了http协议和IIS处理，这次说下当IIS把请求交给Asp.net后，后续的过程。首先要确定IIS是如何调用Asp.net的程序的。       
#### AppManagerAppDomain  

 1. 当IIS把请求交给asp.net时候，如果AppDomain还不存在则创建APPDomain，将AppDomain指派给与请求对应的应用程序，这通过AppManagerAppDomain类中的Create方法实现，代码如下：   
 
``` C#     
  public object Create(string appId, string appPath)
        {
            object result;
            try
            {
                if (appPath[0] == '.')
                {
                    FileInfo fileInfo = new FileInfo(appPath);
                    appPath = fileInfo.FullName;
                }
                if (!StringUtil.StringEndsWith(appPath, '\\'))
                {
                    appPath += "\\";
                }
                ISAPIApplicationHost appHost = new ISAPIApplicationHost(appId, appPath, false);
                ISAPIRuntime iSAPIRuntime = (ISAPIRuntime)this._appManager.CreateObjectInternal(appId, typeof(ISAPIRuntime), appHost, false, null);
                iSAPIRuntime.StartProcessing();
                result = new ObjectHandle(iSAPIRuntime);
            }
            catch (Exception)
            {
                throw;
            }
            return result;
        }

```      
 
#### ISAPIRuntime    



