The packaging for this project did not assign a file to the build artifact
```
mvn clean install
mvn clean install:install
```

前者会运行编译、打包、测试、安装等，每个周期中运行所有目标。后者甚至不会编译或打包代码，它只会运行 install 目标