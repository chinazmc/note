
[TOC]

### 文档信息
####创建信息
|文档编号|文档名称|作者|创建时间|
|---|---|---|---|
|Hyx-R&d-20160318|git分支管理规范|rose|2016-03-18|
####修改记录
|修改时间|修改人|修改备注|
|---|---|---|
|2016-08-12|rose|调整分支命名规范|

## 1.Git Flow 工作原理
>
### 1.1 分支管理
>![图片标题](http://doc.maizuo.com/api/file/getImage?fileId=5870a292d59f44000a00019a)
>首先，项目存在两个长期分支:
> master分支----发布主分支
> develop分支----开发主分支
> 其中master上永远都是稳定的发布版本代码，develop上存放着最新的开发版本
代码(其中,最初的develop分支应该是拉取master生成而来)
>此外，项目存在三种短期分支:
> feature分支----功能分支 
> hotfix分支----补丁分支
> release分支---- 预发布分支
>   其中 ，hotfix分支是直接拉取master分支建立而成，主要是为了快速修复在生产环境遇到的bug而建立的补丁分支，在修改完bug后需要分别合并到master和develop分支上，并且把自己删除掉(这一点gitflow的插件，即git的增强命令集会自动帮我们完成这一点),并且在提交之前会打上标签,feature分支是在develop的基础上checkout了新分支，主
要用于开发，其命名规则 feature/blog_builder;分支都必须加上前缀feature/，并且在开发完成后将代码合并到develop分支并删除自己；release分支在每次确认需要发布时由
develop分支拉建立，其命名release/v5.0前缀为:release/ + 版本号的命名规则，并且打
上标签为v5.0之类，接着分别合并到master和develop分支上


### 1.2 各个分支见得工作流程及状态

master分支（1个）

develop分支（每个环境1个）

feature分支。同时存在多个，用于业务开发

release分支。同一时间只有1个，生命周期很短，只是为了发布  //暂不启用

hotfix分支。同一时间只有1个。生命周期较短，用了修复bug或小粒度修改发布。

>    其中，master和develop都具有象征意义。master分支上的代码总是稳定的（stable build），随时可以发布出去。develop上的代码总是从feature上合并过来的，可以进行Nightly Builds，但不直接在develop上进行开发。当develop上的feature足够多以至于可以进行新版本的发布时，可以创建release分支。
>    release分支基于develop，进行很简单的修改后就被合并到master，并打上tag，表示可以发布了,紧接着release将被合并到develop；此时develop可能往前跑了一段，出现合并冲突，需要手工解决冲突后再次合并。这步完成后就删除release分支。
>    当从已发布版本中发现bug要修复时，就应用到hotfix分支了。hotfix基于master分支，完成bug修复或紧急修改后，要merge回master，打上一个新的tag，并merge回develop，删除hotfix分支。

>   由此可见release和hotfix的生命周期都较短，master/develop虽然总是存在但却不常使用。

>    feature总是基于某一时刻的develop建立而成，其实也是我们工作中的开发分支了，主要是自己做自己的事情咯，差不多的时候要合并回develop去。从不与master交互。
>    develop 主要是和feature以及release交互
>    release  总是基于develop，最后又合并回develop。当然对应的tag跑到master这边去了
>    hotfix   总是基于master，并最后合并到master和develop。
>    master   仅是一些关联的tag，因从不在master上开发,用来做稳定版本的发布
>   其中，分支命名: release-* 或 release/*

### 1.3 标签规范
> 由于在gitflow中tag 是在 release 确认最终需要合到master和develop上做发布的，hotfix是做线上master分支的紧急修复的；由gitflow自行根据release/v5.0这样的分支命名自行获取v5.0或hotfix/v5.1做为bug 修复分支的v5.1 来做为tag的
> 

## 2 分支管理规范(重点且强制执行）
### 2.1分支命名规范
> 分支命名只能采用数字，小写英文字母，下划线,小圆点，长度尽量不要超过30个字符
> 例如  feature/app5.0  
>       hotfix/app5.0  
>       feature/app5.0-order
### 2.2结合公司目前的分支管理情况做的一些建议
>  基于目前公司都是用develop分支发布，并建立新分支做开发的！可以借鉴gitflow的一些流程做到规范点的git分支管理
### 2.3 主分支

> <font color=#FF8C69> master（生产环境最稳定版本分支） 
>  deploy-prod (发布生产环境的主分支) 
>  deploy-stage（发布预生产环境的主分支）
>  deploy-dev/deploy-ali（发布本地开发环境的主分支）
>  deploy-dev/deploy-ali，deploy-stage 和 deploy-prod 为主要管理分支，多人协助的项目尤为重要。
</font>
### 2.4 其他分支
>  <font color=#FF8C69> 开发分支   基于master主分支创建开发分支feature/xxx，例如(feature/app5.0)
>  补丁分支   针对某个版本的修订分支hotfix/xxx
</font>
### 2.5 操作流程
> <font color=#FF8C69>基于master主分支创建命名为feature/xxx开发分支,命名最好包含项目版本号。 
> 严禁频繁提交代码,例如每开发几行代码就推送到远端服务器。 
> 代码提交请详细书写提交描述。 
> 本地环境发布：管理员(多人协助项目指定管理员)将当前开发分支合并到到deploy-dev/deploy-ali分支进行测试，一旦出现代码冲突，召集相关人员沟通解决方案。 
> 预生产环境发布：本地环境测试通过了之后feature/xxx分支合到deploy-stage分支发布接着测试。 
> 生产环境发布：预生产环境测试通过后feature/xxx分支代码合并deploy-prod主开发分支，同一发布周期内，不允许不同版本的分支合并到deploy-prod主开发分支；
> 生产环境发布分支测试完毕，控制在6个小时内将feature/xxx分支代码合并到master分支，打上tag为版本号标签 git tag -a v1.4(版本号) -m 
‘标签的描述信息’ ，先然后把本地master分支和tag分别push到remote
> 开发分支清理 管理员不定期删除掉本地无效feature分支，同时将空分支推送到远端 git push origin :feature/xxx, 
通知项目的其他同事也清理本地分支，然后执行git fetch -p 。 

</font>
### 2.6 参考资料
  - https://git-scm.com/      git官网
  - http://www.runoob.com/git/git-tutorial.html  git教程
  - http://www.ituring.com.cn/article/56870 图林社区
  - http://blog.jobbole.com/76867/          伯乐在线
  - https://github.com/nvie/gitflow/        github
  - https://github.com/git/git/             

