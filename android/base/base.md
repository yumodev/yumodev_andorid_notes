### android工程的目录结构：

   src：工程源代码
   gen：自动生成的内容，主要是资源的ID，等等。
   assets：存放一些随工程打包的文件，主要是图片，文件等等。
   bin：编译过程中产生的一些文件。
   libs：存放第三方的jar包。
   res：存放资源文件的
   AndroidManifest.xml:整个Android项目的配置文件。
   project.properties:制定一个android工程编译时 使用的版本等等。

### dp、sp、px、pt

   px: 像素
   pt：磅，1磅等于1/72英寸。
   dp：密度无关的像素。在屏幕密度为160dp时，1dp等于1px。
   sp：可伸缩像素。与密度无关，也和scale无关。

   如果屏幕为160，那么sp,dp、px是一样的。

### Manifest.xml文件主要包含什么信息？

   * manifest：根节点，描述package中所有的内容
   * uses－permission：请求你的package正常运走所需赋予的安全许可。
   * permission：声明了安全许可限制哪些程序能使用package中的组件和功能。
   * instrumentation：声明了用来测试次package或其他package指令组件的代码。
   * application：包含package中applicaiton级别组件声明的根结点。
   * activity：
   * receiver：
   * service：
   * provider

   ### 系统相关

   * 哪些属性可以当做设置唯一ID

      1. deviceId
      2. mac Address
      3. Sim Serical Number
      4. ANDROID_ID
      5. Serical Number
      6. 自己生产一个安装UUID

   * Android中Kill和 Force Stop的区别

      1. kill
      2. Force Stop

   ### 系统数据

   * Android中应用信息页面清除数据

   清除用户配置，SharedPreferences,数据库等等，Data目录下处理lib之外的所有东东。
   /scdcar/android/data/package/下的所有东东

   * 清除缓存
      清除/data/data/<Package>/下的所有东东
      清除/scdcar/android/data/package/下的所有东东的东东
