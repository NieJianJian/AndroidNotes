## Git

• 创建本地仓库

```git init
git init
```

• 将远程仓库拷贝到本地

```
git clone [url]
例如: git clone https://github.com/NieJianJian/AndroidNotes.git
```

• 拉取远程仓库

```
git pull = git fetch + git merge
git pull [url]
例如：git pull https://github.com/NieJianJian/AndroidNotes.git
```

• 创建远程仓库

```
// 添加一个新的remote远程仓库
git remote add [remote-name] [url]
例如 ：git remote add origin https://github.com/NieJianJian/AndroidNotes.git
origin : 相当于远程仓库的别名

// 列出所有 remote 的别名
git remote

// 列出所有 remote 的 url
git remote -v

// 删除一个 renote
git remote rm [name]

// 重命名 remote
git remote rename [old-name] [new-name]
```

• 从本地仓库添加文件

```
git add .  				// 添加所有修改文件
git add [file-name]  	// 添加指定文件
```

• 提交内容，将缓存内容提交到HEAD

```
git commit -m "[注释内容]"
```

• 查看状态

```
git status
// 查看到的文件名为“褐色” ： 即修改的文件，还未执行 git add 操作
// 查看到得文件名为“绿色” ： 即执行过 git add 操作，还未执行 git commit 操作
// 如果执行过git commit 操作，git status去查看时，绿色的也消失了
```

• 放弃更改

```
git checkout -- [file-name] 		// 只能放弃还未执行过git add操作的文件
```

• 查看log

```
git  log    				// 查看提交记录
git  log  -2   				// 查看近两次的提交记录
git  log  [file-name]  		// 查看file文件的提交记录
git  log  [file/]   			// 查看file文件夹下的提交记录
git  reflog   				// 查看所有分之下的提交记录
```

• 撤销

```
// 将执行过git add操作，还未执行git commit 得文件，撤销，这样git status查看状态，文件名将由绿色变回褐色
git reset HEAD [file-name] 			
// 当执行过git commit操作后，就需要通过revert来撤销操作
git revert [HEAD]
执行 git log，获取提交记录的格式如下：
	----------------------------------------------------------------------------
	commit d657d3d90167fe32e3c264af5facc00aeac55     // 这就是[HEAD]
	Author: Niejj <niejian_sir@163.com>
	Date:   Mon Mar 11 22:40:03 2019 +0800
	
  		git		// 这是git commit -m "注释"，对应的是注释内容

	commit b9813a687906c41e58b1f99894b0566ad3e80
	Author: NieJianJian <niejian_sir@163.com>
	Date:   Tue Mar 14 17:25:19 2017 +0800
	
    	Rename Singleton.md to README.md
    ----------------------------------------------------------------------------
可以通过[ENTER] 或者 [向下键]一直查看记录。
退出查看记录 ： q

// 撤销commit操作，执行如下：
git revert d657d3d90167fe32e3c264af5facc00aeac55	
```

• 退出编辑模式

```
git revert 操作会跳转到编辑模式
1.按[ESC]键输入 
2.输入  :wq
3.按下[ENTER]键

:w 					// 保存；
:w [filename] 		// 另存为filename；
:wq! 				// 保存并强制退出；
:wq! filename 		// 以filename为文件名保存后强制退出；
:q! 				// 不保存文件，强制退出；
:x 					// 保存并退出（仅当文件有变化时保存）

w --- 保存
q --- 退出
! --- 强制执行操作
```

• 

• 

• 

• 

• 