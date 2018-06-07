## dev_1.1版本的改动概要

> 基于master的打包后上传，添加了packer打包、提高推送的送达率、项目的结构调整。

### packer打多渠道包、打包上传至fir

 - 首先需要在项目中，接入packer打包的插件，接入的过程参考 [packer-ng-plugin](https://github.com/mcxiaoke/packer-ng-plugin)
 - 接入成功后，在app内新建packer.gradle文件，并添加以下内容:
    ```
	    apply plugin: 'packer'
		packer {
		    // 多渠道批量打包脚本处理（mac）   ./gradlew clean apkRelease  打包命令
		    // 单一渠道批量打包脚本处理(window) gradle clean apkRelease
		    // 多渠道批量打包脚本处理(window) gradle clean apkRelease -Pchannels=@markets.txt
		    archiveNameFormat = '${buildType}-v${versionName}-${channel}'
		    archiveOutput = new File(project.rootProject.buildDir, "apks")
		    channelList = ['xiaomi','meizu']
		    channelFile = new File(project.rootDir, "markets.txt")
		}

    ```
 - 在app中build.gradle文件的顶部引入packer.gradle文件。
    > apply from: "package.gradle"

 - 将之前的app中build.gradle中的task debugToFir任务，提取出来放入根目录中创建upLoad.gradle中，改动后代码如下：
	```
	ext {
	  def channel = rootProject.ext.channel
      def buildstyle = rootProject.ext.buildstyle
      def android_cfg = rootProject.ext.android
      upUrl = "http://api.fir.im/apps"
	  appName = "androidstduy"
	  bundleId = android_cfg.applicationId
	  verName = android_cfg.versionName
      apiToken = "e8f8a14e9cfd279e27f6b8f81b50163b"
      iconPath = project.rootDir.absolutePath+"/app/src/main/res/mipmap-xxhdpi/app_logo.png"
	  apkPath  = project.rootDir.absolutePath+"/build/apks/"+"${buildstyle[1]}-v${android_cfg.versionName}-${channel[0]}"+".apk"
	  buildNumber = android_cfg.versionCode
	  changeLog = "渠道标识："+channel[0]
	  startUpload = this.&startUpload
	}

	//上传至fir
	def startUpload() {
	  println("开始上传至fir")
	//    def process = "python upToFir.py ${upUrl} ${appName} ${bundleId} ${verName} ${apiToken} ${iconPath} ${apkPath} ${buildNumber} ${changeLog}".execute()
	  def process = "python upToFir.py $upUrl $appName $bundleId $verName $apiToken $iconPath $apkPath $buildNumber $changeLog".execute()
	// 获取Python脚本日志，便于出错调试
	  ByteArrayOutputStream result = new ByteArrayOutputStream()
	  def inputStream = process.getInputStream()
	  byte[] buffer = new byte[1024]
	  int length
	  while ((length = inputStream.read(buffer)) != -1) {
		 result.write(buffer, 0, length)
	  }
	  println(result.toString("UTF-8"))
	  println "上传结束 "
	}
	```

    > 并将upLoad.gradle文件添加至根目录的build.gradle文件，修改为：
    ```
	apply from :'config.gradle'
	apply from: "upload.gradle"
      ```
 - 最后，在app中bulid.gradle文件中创建新的task任务，如下：
   ```
	 def startUpload = rootProject.ext.startUpload
	 task toFir {
		  //依赖执行 --> gradle apkRelease
		  dependsOn 'apkRelease'
		  //doLast 在最后执行，
		  doLast {
		      //上传apk文件到Fir
			  startUpload()
		  }
	 }
   ```
- 在studio的terminal窗口，执行命令，就可以完成打包-->上传文件到Fir了。
  > gradle task toFir

### 提高推送的送达率
> 为了提高推送的送达率，建议用户 允许app手机通知、开机自启动、加入白名单等功能。这里提供实现方式，具体操作表现形式，建议参考 **企业微信的消息推送的引导**
- 实现方式在我的另一篇文章做了详细的介绍：[**Android权限之通知、自启动跳转**](https://github.com/zylaoshi/AndroidTotal/blob/master/Android%E6%9D%83%E9%99%90%E4%B9%8B%E9%80%9A%E7%9F%A5%E3%80%81%E8%87%AA%E5%90%AF%E5%8A%A8%E8%B7%B3%E8%BD%AC.md)

- 代码部分可以在项目中的 [StartUtils](https://github.com/zylaoshi/androidstudy/blob/dev_1.1/app/src/main/java/com/wljr/androidstudy/util/StartUtils.java) 工具类中查看。

### 项目的结构调整
> 主要将项目app中的build.gradle文件中的一些内容解耦，使之不再臃肿。具体实现详见工程代码。