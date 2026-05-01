# 手机obsidian爆改备忘录

项目介绍：将手机版obsidian爆改备忘录，效果类似于下图：

<img src="http://oss.tilex.world/d/blog-images/2026/05/01/image-20260501150420706.png?sign=4CWP3a6us5lkZLjE_4Z-YX8bZ5JkbSzV622eiQRFkhI=:0" alt="image-20260501150420706" style="zoom:50%;" />

点击右下角的按钮，即可添加信息

## 环境准备

首先需要安装 Obsidian 手机版本，并创建一个本地仓库作为数据存储空间

然后将项目中的 `随记` 文件夹复制到仓库目录中，这个文件夹已经包含了基础结构与示例数据，可以直接使用

目录结构核心如下：

```
随记/
├── index.md          // 首页
├── messages/         // 消息数据存储
├── setting/          // 配置（头像等）
```

## 插件配置

为了实现完整功能，需要安装并启用三个插件。

### HomePage：构建入口页面

该插件用于指定启动页面，让 Obsidian 打开即进入备忘录界面。

配置方式：

- 启用插件
- 设置启动页为：

```
随记/index
```

这样每次打开应用时，直接进入备忘录主界面，而不是文件列表。

### Banners：增强视觉表现

Banners 用于为页面添加头图，使界面更接近现代应用风格。

但在移动端存在一个问题：头图通过“笔记属性”控制，而属性默认会显示在页面上，影响视觉效果。

解决方式：

- 启用 Banners 插件
- 在设置中关闭“显示笔记属性”

这样既保留头图，又不会干扰界面布局。

<img src="http://oss.tilex.world/d/blog-images/2026/05/01/image-20260501152943529.png?sign=1aAfxqG8J_lxOYPitl85dpu33aU87_eFzw86SgLNcYQ=:0" alt="image-20260501152943529" style="zoom:50%;" />

### DataView：实现动态渲染

这是整个项目的核心插件；通过 DataViewJS，可以动态读取 `messages` 文件夹中的内容，并按照时间排序渲染成“消息流”，下载 DataView 插件后开启下图两项功能

<img src="http://oss.tilex.world/d/blog-images/2026/05/01/image-20260501153404665.png?sign=FSeEZXE-18Ine6PeLE6CJ0mcQUKzBjWn90Vh91nSZUk=:0" alt="image-20260501153404665" style="zoom:50%;" />

## 样式系统：让界面更像应用

默认的 Markdown 渲染并不适合做“聊天流”，因此需要自定义 CSS。

将 `chat.css` 放入：

```
.obsidian/snippets/
```

然后在设置中启用

<img src="http://oss.tilex.world/d/blog-images/2026/05/01/image-20260501153847097.png?sign=qwCXpJSTT0u62AcgM6K0Qwdvt-n3oIEjFhtymHfnyLk=:0" alt="image-20260501153847097" style="zoom:50%;" />

完成后，你将获得一个具备以下能力的系统：

- 打开即进入备忘录首页
- 顶部有视觉头图
- 中间是按时间排序的消息流
- 每条记录以“聊天气泡”形式展示
- 支持快速新增内容































