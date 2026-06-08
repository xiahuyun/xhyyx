
## 文章分类

- **Go 语言**: Go Runtime Scheduler、锁机制、Plan9 汇编、net/http
- **Kubernetes**: Kube-apiserver、Kubelet、Client-go、Scheduler
- **Containerd**: 源码分析、启动流程、与 Kubelet 交互
- **Raft**: 一致性算法读写流程
- **Go-zero**: RESTful API 框架

## 配置说明

### 主要配置项

| 配置项 | 说明 |
|--------|------|
| `baseURL` | 站点基础 URL |
| `title` | 站点标题 |
| `author` | 作者名称 |
| `defaultTheme` | 默认主题 (dark/light) |
| `ShowToc` | 是否显示文章目录 |
| `ShowCodeCopyButtons` | 是否显示代码复制按钮 |

### 社交图标

在 `hugo.yaml` 中配置社交链接：

```yaml
socialIcons:
  - name: github
    url: "https://github.com/xiahuyun"
  - name: rss
    url: "https://www.cnblogs.com/xingzheanan"
  - name: wechat
    url: "https://mp.weixin.qq.com/s/JcewFRttKz31sPsnqdeBOw"
```

## 部署

项目已配置自动部署到 Cloudflare Pages，每次推送到主分支自动触发构建。

## License

MIT License

## 致谢

- [Hugo](https://gohugo.io/) - 静态站点生成器
- [PaperMod](https://github.com/adityatelange/hugo-PaperMod) - Hugo 主题
- [Giscus](https://giscus.app/) - GitHub Discussions 评论系统