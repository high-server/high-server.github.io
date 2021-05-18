# 新建文章步骤
- 1.创建新文章到恰当目录下，例如

```sh
#hugo new post/inside-golang/title.md
```

- 2.编辑新博客文件
```
#vi content/post/inside-golang/title.md
```

- 3.本地测试运行
```
#hugo server --buildDrafts
```

- 4.编译静态页面到docs目录下
```
#hugo
```

- 5.发布
```
#git push origin main
```