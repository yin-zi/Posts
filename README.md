**使用说明**

1. [下载](https://github.com/gohugoio/hugo/releases/latest) `hugo_extended` (扩展版有增强图片处理)，放到项目根路径即可(或者任意路径+环境变量)

   > 官方文档 包含其它操作系统 [Installation | Hugo (gohugo.io)](https://gohugo.io/installation/)

2. 使用命令 `hugo new content posts/[dir/]<xxx.md> ` 新建文档

   > 请注意文档要生成在 `content/posts` 及其子目录中

3. 编写完成后，需要将文档头中的 `draft` 值改为 `false`

   > 此字段表明文档是否是草稿(即是否写完)，只有写完的文档才会展示

4. 可以使用 `hugo server` 本地开启服务查看

   > 启动后会自动监听文件修改

5. 使用 `git` 正常提交即可，请注意不要提交其它无用文件

6. 当 `main` 分支有新提交自动触发构建部署

**有关图片的说明**

- 本仓库 `main-picBed` 分支即为图床分支
- 可使用 `typora` 中 图像上传服务的 `PicGo-Core` 功能，配置如下

  > 具体操作typora有文档

```json
{
  "picBed": {
    "uploader": "github",
    "current": "github",
    "github": {
      "repo": "yin-zi/Posts",
      "branch": "main-picBed",
      "token": "************************",
      "path": "static/images",
      "customUrl": ""
    },
    "transformer": "path"
  },
  "picgoPlugins": {}
}
```

**其它**

- `hugo` [Quick start | Hugo (gohugo.io)](https://gohugo.io/getting-started/quick-start/)

