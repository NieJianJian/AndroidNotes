## Maven相关知识

1. 解压aar

   ```
   unzip AMap_xxxx.aar -d temp
   ```

2. 打包aar

   ```
   jar cvf opensdk.aar -C temp/ ./
   ```

3. 解压后删除不用的资源

   * 删除class文件

     ```
     zip -d classes.jar 'com/alibaba/fastjson/*'
     ```

   * 删除so文件：

     ```
     直接在目录文件夹下删除即可
     ```