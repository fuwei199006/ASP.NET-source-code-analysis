上次说到获得HttpApplication对象的创建，创建完成后调用InitInternal方法，下面详细过程。    

###InitInternal    

在这个方法任务比较多，也比较长，这里就不贴全码了，一个一个过程的去说：       

#### 初始化HttpModule      

##### httpModule使用实例  

1. HttpModule在Asp.net中有重要的作用，扩展起来也是非常的简单。  



#### Global内的方法调用     


#### 根据应用程序池的类型创建不同的_stepManager      


#### 执行BuildStep