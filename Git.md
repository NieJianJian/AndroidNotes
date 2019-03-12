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

• 

• 

• 