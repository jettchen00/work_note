# 1. 源码编译安装UE4
1. 需要将github账号和EpicGames账号关联起来，才能下载源码。登录EpicGames官网，进入个人信息页面，找到连接、账号，然后连接github账号。
2. 先安装visual studio 2017或者visual studio 2019，下载好UE4代码，在资源管理器中打开你的源代码文件夹，并运行 Setup.bat 。运行 GenerateProjectFiles.bat 来为引擎创建项目文件。双击 UE4.sln 文件以将项目加载到Visual Studio中。将你的解决方案配置设置为 开发编辑器 ，将解决方案平台设置为 Win64 ，然后右键单击 UE4 目标并选择 构建 。之后，在UnrealEngine\Engine\Binaries\Win64目录中，就生成了可执行文件UE4Editor.exe。