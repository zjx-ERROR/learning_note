# git_rm_large_objects

1. 拉取项目：将项目所有分支拉下来
```bash
git clone yourproject
cd yourproject
git pull --all
```

查看删除前大小（仅做对比）
```bash
git count-objects -vH
```

2. 查找大文件
```bash
git rev-list --objects --all | grep "$(git verify-pack -v .git/objects/pack/*.idx | sort -k 3 -n | tail -10 | awk '{print$1}')"
```

3. 删除大文件
```bash
git filter-branch --force --index-filter "git rm -r --cached --ignore-unmatch yourfile" --tag-name-filter cat -- --all
```

删除每个commit中包含的文件，出现rm表示该commit包含文件同时删除成功

同时回收本地空间
```bash
git for-each-ref --format='delete %(refname)' refs/original | git update-ref --stdin && git reflog expire --expire=now --all|git gc --prune=now
```

查看删除后大小
```bash
git count-objects -vH
```

4. 强制推送到服务器（分支需要取消保护）
```bash
git push origin --force "refs/heads/*" --tags
```

5. 清除服务缓存

进入git服务器，进入项目文件夹，执行清理无效文件
```bash
git gc --prune=now
```
此时git仓库就减小了