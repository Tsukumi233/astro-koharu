---
title: 给hexo-next博客添加石蒜摇摇乐
link: gei-hexo-next-bo-ke-tian-jia-shi-suan-yao-yao-le
date: 2023-12-31
tags:
  - 博客
  - hexo
  - next
categories:
  - 技术
---

在\themes\next\ _config.yml 中将以下的 footer 条目取消注释

```plain
custom_file_path:
    footer: source/_data/footer.swig
```
然后在\source\ _data\footer.swig 文件（没有则新建）中粘贴以下内容
（代码来源：https://github.com/itorr/sakana）

```plain
<meta name="viewport" content="width=device-width">
<style>

html .sakana-box{
  position: fixed;
  right: 0;
  bottom: 0;
  
  transform-origin: 100% 100%; /* 从右下开始变换 */
}
</style>

<div class="sakana-box"></div>

<script src="https://cdn.jsdelivr.net/npm/sakana@1.0.8"></script>
<script>
// 取消静音
Sakana.setMute(false);

// 启动
Sakana.init({
  el:         '.sakana-box',     // 启动元素 node 或 选择器
  scale:      .5,                // 缩放倍数
  canSwitchCharacter: true,      // 允许换角色
});
</script>
```

hexo clean && hexo g && hexo s 之后就可以愉快地玩摇摇乐了~