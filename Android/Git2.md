## git

* commit修改

  ```
  git commit --amend
  ```

  进入输入i进入编辑模式、编辑，esc、`:wq!`保存。

* 撤销修改

  ```
  git checkout -- [file-name]
  ```

* 撤销commit

  ```
  git reset --soft HEAD^  // 回到commit之前，add之后
  git reset --soft [head] // 回到head对应的提交，之后的提交都撤销
  ```

  HEAD^的意思是上一个版本，也可以写成HEAD~1

  如果你进行了2次commit，想都撤回，可以使用HEAD~2

  `--mixed ` ：撤销commit，并且撤销git add 操作。默认参数

  `--soft  `：不删除工作空间改动代码，撤销commit，不撤销git add . 

  `--hard`：删除工作空间改动代码，撤销commit，撤销git add . 

  * 如果commit已经push到远程仓库，reset之后执行`git psuh`会提示本地版本落后，无法成功。所以需要强制push

    ```
    git push --fopuysrce   // 简写 -f 也行--
    ```

* 撤销add

  ```
  git reset HEAD [file-name] 
  git reset HEAD  // 后面什么都不加，就是撤销所有执行了add的文件
  ```

* 删除没有add的新增文件

  ```
  git clean 参数
      -n 显示 将要 删除的 文件 和  目录
      -f 删除 文件，-df 删除 文件 和 目录
  ```

* log

  ```
  git log --oneline -5
  ```

  查看指定分支log

  ```
  git log [branch]
  ```

  查看所有分支图形化的commit历史(oneline 一条提交信息用一行展示)

  ```
  git log --all --oneline --graph
  ```

  ```
  git log --graph --pretty=oneline --abbrev-commit
  ```

* merge

  * dev合并到master分支

    ```
    git checkout master
    git merge dev
    ```

* 分支

  * 查看本地分支

    ```
    git branch
    ```

  * 查看本地分支和远程分支

    ```
    git branch -a
    ```

  * 切换分支

    ```  
    git checkout [branch-name]
    git switch -c [branch-name]   // -c 是创建并切换
    git checkout [head] -b [branch-name]   // 从某次提交拉取一个分支
    ```

  * 拉取远程分支到本地

    ```
    git checkout -b [name] [remote-name]
    ```
    
  * 删除分支

    ```
    git branch -d [branch-name]
    git branch -D [branch-name]  // 强制删除
    ```

* cherry-pick

  dev分支的某个提交合并到master分支

  ```
  git log // dev分支查看提交log，找到对应的head哈希值
  git checkout master // 切换到master分支
  git cherry-pick [head]  // 将dev分支的[head]提交添加到master分支
  ```

  不一定使用哈希值，也可以转移分支最新的提交

  ```
  git cherry-pick dev  // dev分支的最新一次提交就到了master下
  ```

  转移多个提交

  ```
  git cherry-pick [headA] [headB]  // 将A、B两个提交转移到master
  ```

  转移一系列提交

  ```
  git cherry-pick A..B   // A到B的提交，不包含A，A必须早于B提交
  git cherry-pick A^..B   // 包含A
  ```

  一般会遇到冲突，解决冲突

  ```
  git add
  git cherry-pick --continue
  解决下一个冲突
  git add
  git cherry-pick --continue
  ...
  ```

  [git cherry-pick 教程](http://www.ruanyifeng.com/blog/2020/04/git-cherry-pick.html)

  [[Git] Git整理(五) git cherry-pick的使用](https://blog.csdn.net/fightfightfight/article/details/81039050)

* stash

  * 切换别的分支前，暂存当前修改

    ```
    git stash
    ```

    或者添加注释信息

    ```
    git stash sava "注释信息"
    ```

  * 当切换会当前分支，需要恢复修改

    ```
    git stash pop
    ```

    或者执行某一个暂存

    ```
    git stash list
    git stash apply stash@{0}
    ```

  * 查看stash内容

    ```
    git stash show
    git stash show -p 
    git stash show -p stash@{1}
    ```

  * 移除stash

    ```
    git stash drop stash@{0}
    ```
  
* tag

  * 查看tag

    ```
    git tag
    git tag -l "v1.0" // 指定version的tag
  git tag -l "v1.*" // 通配符
    ```
    
  * 添加tag

    ```
    git tag -a "v1.0.0" -m "this is version 1.0.0"
    ```

    `-m` 选项指定了一条将会存储在标签中的信息。 如果没有为附注标签指定一条信息，Git 会启动编辑器要求你输入信息。

  * 添加轻量tag

    ```
    git tag v1.2
    ```

  * 查看tag和对应的提交信息

    ```
    git show v1.0
    ```

  * 提交tag

    ```
    git push origin v1.2			// 推送本地执行版本号的tag
    git push origin --tags   // 推送本地所有tag
    ```

    

***

### Git规范

[如何规范你的Git commit？](https://blog.csdn.net/amap_tech/article/details/108289242)

* **commit message格式**

  ```
  <type>(<scope>): <subject>
  ```

* **type(必须)**

  用于说明git commit的类别，只允许使用下面的标识。

  feat：新功能（feature）。

  fix/to：修复bug，可以是QA发现的BUG，也可以是研发自己发现的BUG。

  - fix：产生diff并自动修复此问题。适合于一次提交直接修复问题

  - to：只产生diff不自动修复此问题。适合于多次提交。最终修复问题提交时使用fix

  docs：文档（documentation）。

  style：格式（不影响代码运行的变动）。

  refactor：重构（即不是新增功能，也不是修改bug的代码变动）。

  perf：优化相关，比如提升性能、体验。

  test：增加测试。

  chore：构建过程或辅助工具的变动。

  revert：回滚到上一个版本。

  merge：代码合并。

  sync：同步主线或分支的Bug。

* 