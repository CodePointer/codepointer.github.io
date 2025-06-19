# Database deployment

Database: Azure Database for PostgreSQL Flexible Server
- Free for 12 months
- Burstable B1MS, 32GB
- 25$/month after free period

按照pdf里面的方式进行录入即可。其中要记得username和密码，这个会比较重要。

而后，在需要后端访问的地方，把DATABASEURL设置好。记得会需要设置好使用SSL访问。

- 管理后端，可以使用pgAdmin，也可以使用vscode的postgresql，但需要记得开启SSL，否则会报no encryption的错误。
- vscode中postgresql的小坑：需要指定database的名字，这个连接默认只连接某一个库。

# 后端 deployment

Backend: Azure Functions + Azure App Service Free (F1)

创建Azure Function App可以参考pdf，一项项来。

考虑使用Azure Function App的主要原因是它对于离散访问更友好一点，免费额度更大。

建议开启Azure的log，方便进行debug。


1. 使用Github的yml部署

Github部分可以直接deploy文件目录，但需要指定Azure进行remote-build。


后端部署Function的时候非常大的坑：

总出现500报错。Log此时报错是FastAPI的module没有安装，即所有的依赖都没有成功安装。研究发现，Github的workflow没有安装，同时remote-build也没有打开。

参考网址：https://stackoverflow.com/questions/79513001/successful-github-actions-workflow-does-not-deploy-python-app-to-azure-function

设置里面一定要明确，不要写什么enable_oryx_build，v1不会支持，github会直接忽略。
同时有一些说法是设置SCM_DO_BUILD_DURING_DEPLOYMENT和ENABLE_ORYX_BUILD，这个也不靠谱，这两个App Setting在Flex SKU里不被支持，Kudu验证阶段就会被打回来，github action会直接报错：

InvalidAppSettingsException: The app settings SCM_DO_BUILD_DURING_DEPLOYMENT, ENABLE_ORYX_BUILD are not supported with this SKU. Please remove the listed app settings and then redeploy.

就按照我的yml写。

成功部署之后，就可以使用了。此时可以直接通过命令行方式进行访问，但很快会发现浏览器访问不了。

需要注意的是其中CORS设置在FastAPI中没有效果，Azure Function App会有自己的CORS，需要开启（我这头哪怕不开也会直接block住）。


# 前端

Azure Static Web Apps Free

注意，尽管我倾向于使用Azure来管理很多环境变量，但对于前端这里环境变量需要由Github来管理。这是由于打包build的过程中它会直接把环境变量注入进去，而Azure那头拿到的是打包后的代码，环境变量用不了。

记得，yml文件必须是Static Web App给的那个yml，不能改名。我知道这很蠢，但这玩意从21年开始一直到现在都没法改。

最后，app_location那个变量要设置成输出的目录。当然，要记得设置好CORS。


# 域名管理

https://sarakale.top/blog/posts/3a730d01/
