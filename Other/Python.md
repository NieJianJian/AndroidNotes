## Python 相关命令

* python安装路径

  * 系统

    ```
    Library/Frameworks/Python.framework/Version/2.7
    ```

  * brew安装

    ```
    /usr/local/Cellar/python@3.9/3.9.1_6/Frameworks/Python.framework/Versions/3.9
    ```

* 查询python脚本安装地址

  ```
  which python/python3
  ```

* 查询python包安装地址

  ```
  python/python3  // 进入python命令行
  import sys
  print(sys.path)/print sys.path
  exit() // 退出python命令行
  ```

  ```
  python/python3 -v   // 小写v，大写V是版本号
  ```

* 查看已经安装的python3

  ```
  brew search python3
  ```

* 查看python信息

  ```
  brew info python3
  ```

* -m

  ```
  python -m scrapy
  python3.9 -m scrapy
  ```

* 下载超时

  * 设置连接超时时间

    ```
    pip --default-timeout=100 install -U 包名
    ```

  * 使用镜像（极快）

    ```
    pip install -i https://pypi.tuna.tsinghua.edu.cn/simple 包名
    ```

* pip下载指定版本

  ```
  pip3.9 install 包名==version
  ```

  