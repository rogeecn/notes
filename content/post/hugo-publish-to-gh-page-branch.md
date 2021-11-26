---
title: "Hugo 发布到 gh-pages 分支"
date: 2021-11-26T12:16:38+08:00
draft: false
tags: ["Hugo", "Github"]
categories: ["网站开发"]

---
如果希望单独控制源文档和生成的网页文档的版本历史，可以使用单独建立一个gh-pages branch的方法托管到Github Pages——先将/public子目录添加到.gitignore中，让master branch忽略其更新，然后在本地和Github端添加一个名为gh-pages的branch：

```bash
# 忽略public子目录
echo "public" >> .gitignore
# 初始化gh-pages branch
git checkout --orphan gh-pages
git reset --hard
git commit --allow-empty -m "Initializing gh-pages branch"
git push origin gh-pages
git checkout main
```

为了提高每次发布的效率，可以将下述命令存在脚本中，每次只需要运行该脚本即可将gh-pages branch中的文章发布到Github的repo中：

```bash
#!/bin/sh

if [[ $(git status -s) ]]
then
    echo "The working directory is dirty. Please commit any pending changes."
    exit 1;
fi

echo "Deleting old publication"
rm -rf public
mkdir public
rm -rf .git/worktrees/public/

echo "Checking out gh-pages branch into public"
git worktree add -B gh-pages public origin/gh-pages

echo "Removing existing files"
rm -rf public/*

echo "Generating site"
hugo

echo "Updating gh-pages branch"
cd public && git add --all && git commit -m "Publishing to gh-pages (publish.sh)"

echo "Push to origin"
git push origin gh-pages
```

后将 main branch 中的源文档和gh-pages branch中的网页文档分别push到Github repo中，进入Settings标签菜单，选择Github Pages项中的Source栏，点gh-pages branch选项, 等待片刻，即可访问 https://your_name.github.io 看到用Hugo生成的网页了。
