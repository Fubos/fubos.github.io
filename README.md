# 个人主页（Hugo + hugo-PaperMod + Github Pages + Github Actions)
这是我的[个人主页](https://blog.fun-x.top/)

在[hugo-PaperMod](https://github.com/adityatelange/hugo-PaperMod/) 的基础上进行了一些个人定制化，添加了相册、代码块折叠、目录侧边展示等小功能。

并结合Github Actions实现自动化文章推送以及部署展示。

## 简易相册功能
```html
{{ define "main" }}
<div class="post">
  <h2 class="post-title">{{ .Title }}</h2>
</div>

<div class="page-photos">
  {{ $photoDirs := readDir "./static/photos" }} <!-- 读取 photos 目录中的所有文件夹 -->
  
  {{ $sortedDirs := slice }} <!-- 初始化一个空的 slice 用于存储排序后的目录 -->
  {{ range $photoDirs }}
    {{ if .IsDir }}
      {{ $sortedDirs = $sortedDirs | append . }}
    {{ end }}
  {{ end }}

  <!-- 自定义排序，将目录按时间降序排列 -->
  {{ $sortedDirs = sort $sortedDirs "Name" "desc" }}

  {{ range $sortedDirs }}
    {{ $dirName := .Name }}
    <div class="photo-group">
      <h3 class="group-title">{{ $dirName }}</h3> <!-- 显示日期或作者名称 -->

      <div class="group-photos">
        {{ $photos := readDir (printf "./static/photos/%s" $dirName) }} <!-- 获取每个目录中的图片文件 -->
        {{ range $photos }}
          <figure>
            <!-- 给每个图片包裹<a>标签，使用data-fancybox来触发放大效果 -->
            <a data-fancybox="gallery" href="{{ printf "/photos/%s/%s" $dirName .Name }}" data-caption="{{ .Name }}">
              <img src="{{ printf "/photos/%s/%s" $dirName .Name }}" alt="{{ .Name }}" />
            </a>
            <!-- <figcaption>{{ .Name | replaceRE "(.*)[.].*" "$1" }}</figcaption> -->
          </figure>
        {{ end }}
      </div>
    </div>
  {{ end }}
</div>
{{ end }}

```

实际效果见:[个人主页-photos](https://blog.fun-x.top/posts/photos/)

## 代码块折叠
```html
<details>
    <summary>{{ (.Get 0) }}</summary>
    {{ .Inner | markdownify }}
</details>
```

使用时的短代码书写：
```html
{{< toggle "点这里折叠" >}}
    这里填具体代码
{{< /toggle >}}

```
实际效果查看主页中[具体文章内容](https://blog.fun-x.top/posts/tech/websocket-connection-management/)。


## 目录侧边展示
参考 https://blog.csdn.net/Xuyiming564445/article/details/122011603

## 搭建链接参考

其他具体搭建细节参考如下,感谢搭建过程中参考过的链接：
> [1] https://github.com/adityatelange/hugo-PaperMod/

> [2] https://zhuanlan.zhihu.com/p/422859066?utm_campaign=shareopn&utm_medium=social&utm_psn=1853241839257276416&utm_source=wechat_session&utm_id=0

> [3] https://www.shaohanyun.top/posts/env/blog_build2/

> [4] https://www.elegantcrazy.com/posts/blog/build-blog-with-github-pages-hugo-and-papermod/

> [5] https://blog.csdn.net/Xuyiming564445/article/details/122011603

> [6] https://lifeislife.cn/posts/hugo%E4%B8%BB%E9%A2%98%E9%85%8D%E7%BD%AE/

> [7] https://kyxie.me/zh/blog/tech/web/papermod/#%e5%9b%be%e5%ba%8a

> [8] https://grer.cn/blog/2022/413a754.html/

> [9] https://blog.yuk7.com/posts/papermod/#%e5%85%ab%e6%b7%bb%e5%8a%a0%e5%9b%be%e7%89%87%e7%81%af%e7%ae%b1%e6%95%88%e6%9e%9c

> [10] https://im.happytoo.cyou/blog-toggle/

