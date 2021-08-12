## Jenkins相关知识

1. 进入服务器

   ```
   sudo sh -  
   输入电脑密码
   ssh -p 22 root@192.168.200.216
   V9HiK#6H36wLiIuu

2. 一般jenkins存放目录

   ```
   // jenkins任务的存放目录
   var/lib/jenkins/workspace
   // jenkins任务，包含日志
   var/lib/jenkins/jobs
   ```

3. 常用的操作命令

   * 查看进程

     ```
     ps aux | grep jenkins

   * 查看磁盘大小

     ```
     df -h // 查看磁盘大小
     df -lh // 查看磁盘大小
     ll -la // 查看文件的详情大小，包含.开头文件
     ll -ll // 查看文件的详情大小，不包含.开头文件
     ll -lh // 查看文件的详情大小，不包含.开头文件，并且会以MB、KB为单位
     du -h --max-depth=1 *  // 查看当前目录下各文件，文件夹的大小，以及下一集目录
     du -h --max-depth=0 * // 只显示当前目录下的文件和文件夹大小
     du -sh // 查看当前目录大小，M为单位
     du -sh [aaa] // 查看aa目录的大小
     du -s [aaa] // 单位kb
     ```

4. 部署jenkins-agent

   1. 拷贝本地文件到服务器

      ```
      scp -r /Users/niejianjian/Downloads/jenkins-agent-daily.zip root@192.168.200.216:~/jenkins-agent-daily/
      ```

   2. 杀掉当前对应进程

      ```
      kill 2544
      ```

   3. 备份原来的jenkins-agent

      ```
      mv aa bb
      cp -r aa bb
      ```

   4. 解压最新的jenkins-agent

      ```
      unzip jenkins-agent-daily.zip -d jenkins-agent
      ```

   5. 运行shell脚本

      ```
      cd jenkins-agent/bin
      sh appctl.sh start
      ```
