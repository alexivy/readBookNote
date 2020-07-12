一些小技巧  

- [windows](#windows)
  - [移动设备作为windows的拓展屏幕。](#移动设备作为windows的拓展屏幕)
- [IDEA](#idea)
  - [导航栏消失](#导航栏消失)
  - [延长试用时间](#延长试用时间)
  - [maven仓库](#maven仓库)

# windows  
## 移动设备作为windows的拓展屏幕。  
推荐spacedesk。i3-4005u nvidia-920M win10-1909工作正常。  
优点：免费，文字显示还ok，支持ios & android，基本流畅。  
缺点：只能无线连接，延迟目测100ms左右，可调设置较少。  
另外就是duet display。ios 68元，不知道好不好用。  


# IDEA  
##  导航栏消失  
如图进行配置：  

![image](/image/20200707001.png)

## 延长试用时间  
脚本内容如下：  
```
file name： reset_jetbrains_eval_windows.vbs

Set oShell = CreateObject("WScript.Shell")
Set oFS = CreateObject("Scripting.FileSystemObject")
sHomeFolder = oShell.ExpandEnvironmentStrings("%USERPROFILE%")
sJBDataFolder = oShell.ExpandEnvironmentStrings("%APPDATA%") + "\JetBrains"

Set re = New RegExp
re.Global     = True
re.IgnoreCase = True
re.Pattern    = "\.?(IntelliJIdea|GoLand|CLion|PyCharm|DataGrip|RubyMine|AppCode|PhpStorm|WebStorm|Rider).*"

Sub removeEval(ByVal file, ByVal sEvalPath)
	bMatch = re.Test(file.Name)
    If Not bMatch Then
		Exit Sub
	End If

	If oFS.FolderExists(sEvalPath) Then
		oFS.DeleteFolder sEvalPath, True 
	End If
End Sub

If oFS.FolderExists(sHomeFolder) Then
	For Each oFile In oFS.GetFolder(sHomeFolder).SubFolders
    	removeEval oFile, sHomeFolder + "\" + oFile.Name + "\config\eval"
	Next
End If

If oFS.FolderExists(sJBDataFolder) Then
	For Each oFile In oFS.GetFolder(sJBDataFolder).SubFolders
	    removeEval oFile, sJBDataFolder + "\" + oFile.Name + "\eval"
	Next
End If

MsgBox "done"
```
 
## maven仓库  
项目pom中添加如下：  
```
<repositories>
    <repository>
      <id>alimaven</id>
      <name>Maven Aliyun Mirror</name>
      <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
      <releases>
        <enabled>true</enabled>
      </releases>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
    </repository>
  </repositories>
  ```