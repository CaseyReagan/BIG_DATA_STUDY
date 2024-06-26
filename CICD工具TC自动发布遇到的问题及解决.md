# CI/CD工具TeamCity自动发布遇到的问题及解决
借鉴内容：https://blog.csdn.net/weixin_44903147/article/details/96291588.   
https://zhuanlan.zhihu.com/p/381514438.

## 总结
+ （1）CI/CD是一种软件开发的工作流自动化管理工具。
+ （2）与TeamCity相比，Jenkins更受欢迎，但是TC支持java和.net。
+ （3）登录服务器的用户权限和默认路径以及环境变量不同的问题。

### 问题背景
2022年6月8日工作中遇到一个与CI/CD工具TeamCity相关的问题，最终解决，事情经过见当日工作记录。见“六月工作记录”.


### 什么是CI/CD工具
CI/CD 是一种通过在应用开发阶段引入自动化来频繁向客户交付应用的方法。CI/CD 的核心概念是持续集成（Continuous Integration）持续交付（Continuous Delivery）持续部署（Continuous Deployment）。   

现在，在DevOps时代，每周，每天，甚至每天多次发布是常态。当SaaS正在占领世界时，尤其如此，您可以轻松地动态更新应用程序，而无需强迫客户下载新组件。很多时候，用户甚至都不会意识到正在发生变化。开发团队通过软件交付流水线（Pipeline）实现自动化，以缩短交付周期，大多数团队都有自动化流程来检查代码并部署到新环境。   

（1）持续集成：   
现代应用开发的目标是让多位开发人员同时处理同一应用的不同功能。但是，如果企业安排在一天内将所有分支源代码合并在一起（称为“合并日”），最终可能造成工作繁琐、耗时，而且需要手动完成。这是因为当一位独立工作的开发人员对应用进行更改时，有可能会与其他开发人员同时进行的更改发生冲突。持续集成的重点是将各个开发人员的工作集合到一个代码仓库中。通常，每天都要进行几次，主要目的是尽早发现集成错误，使团队更加紧密结合，更好地协作。   

（2）持续交付：   
完成 CI 中构建及单元测试和集成测试的自动化流程后，持续交付可自动将已验证的'代码发布到存储库。为了实现高效的持续交付流程，务必要确保 CI 已内置于开发管道。持续交付的目标是拥有一个可随时部署到生产环境的代码库。持续交付的目的是最小化部署或释放过程中固有的摩擦。它的实现通常能够将构建部署的每个步骤自动化，以便任何时刻能够安全地完成代码发布（理想情况下）。   

（3）持续部署：   
对于一个成熟的 CI/CD 管道来说，最后的阶段是持续部署。作为持续交付——自动将生产就绪型构建版本发布到代码存储库——的延伸，持续部署可以自动将应用发布到生产环境。由于在生产之前的管道阶段没有手动门控，因此持续部署在很大程度上都得依赖精心设计的测试自动化。   

### 什么是TeamCity
两个流行的CI / CD工具是Jenkins和TeamCity，它们各自具有自己的独特功能。TeamCity是一款功能强大的持续集成（Continue Integration）工具，包括服务器端和客户端，支持Java，.NET项目开发。TeamCity是由JetBrains创建的基于Java的CI / CD工具，JetBrains是PyCharm，IntelliJ Idea，RubyMine，ReShaper等其他有用工具的生产者（Source）。小型团队可以免费使用TeamCity。提供对.Net框架的支持，并且可以集成到IDE（如Visual Studio和Eclipse）中。


### 当日所遇问题
因为我们的版本提交并不频繁，所以在距离上一次版本发布一个多月之后，用TC进来自动化版本发布时，遇到了无法提交的问题。我们配置的自动化过程有5个Build步骤：
1. Maven Build。
2. Shut Down JAR（旧的运行中的程序）。
3. Rename Old version JAR（保存一个历史版本）。
4. Copy JAR to target folder(新的JAR程序)
5. Start up JAR。（开启新程序）
step log里问题报错在第3步，提示“The process cannot access the file because it is being used by another process.”以及“Step Rename Old Version JAR (Command Line) failed”。

### 解决过程
1. 第3步的报错，实际上意思是现在的程序依然在运行，打开后台程序管理器看到相关java任务依然在跑，所以问题实际上是出在第2步关闭运行中的程序失败。   
2. 继续查看第2步的记录，告警提示“'netstat' is not recognized as an internal or external command,”。   
3. 开始检查问题，新建一个Java Hello World程序携带一段关于netstat的UID获取和shut down代码，对这个JAR尝试进行自动化发布。提示同样的问题和报错。   
4. 搜索报错语句后发现这是一个windows的shell命令，按道理说不会出现此问题，怀疑是环境变量的配置有错。登录搭载程序的windows server服务器检查window的PATH设置，无任何问题，在服务器的CMD窗口执行netstat命令不报错。证明，windows环境变量无问题。
5. 将java代码编译为一个bat脚本放到服务器之中，手动用管理员权限运行此脚本，监视后台程序id，发现脚本正确运行。证明：服务器环境无任务问题，环境变量也没有任何问题。
6. 怀疑是TC上发布代码的用户权限，和登录windows server的用户权限不相同导致有问题，尝试不使用管理员权限在服务器上手动跑bat脚本，当然肯定不能执行关闭后台程序的操作，但是报错是"Acess Denied"，这代表是没管理员权限，但是关闭程序已经通过执行netstat拿到了程序的uid。证明：二者报错不一致，如果是权限问题，应该报"Acess Denied"而不是无法识别"netstat"脚本。
7. （此处略过一系列走弯路检查问题的过程）尝试再次配置环境变量，并且检查了TC发布用户的权限，问题依旧，最后发现可能是TC的用户进入服务器后在windows里的默认路径有问题，目前的路径指向一个TC下存储临时内容的路径。最终尝试在代码之前增加了cd C:\Windows\System32这一路径命令，竟然成功了。

### 遗留问题
之前的版本发布都没有此问题，怀疑是被修改了TC发布用户的进入的默认路径，但是没有找到什么修改的记录，以及没有跟进寻找TC里哪里配置的进入的默认路径。
