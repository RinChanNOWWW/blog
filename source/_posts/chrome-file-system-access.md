---
title: 文件系统访问 API：简化访问本地文件
date: 2020-12-23 14:31:44
tags:
	- Javascript
	- 学习
	- Web
	- Chrome
	- 文章翻译
categories:
	- 编程学习

---

## 原文章：https://web.dev/file-system-access/

## 什么是文件系统访问 API

文件系统访问 API（[The File System Access API](https://wicg.github.io/file-system-access/)，之前又叫 Native File System API，更之前叫 Writeable Files API）使得开发者们能够构建能与用户本地设备文件交互的强大 Web 应用，比如 IDE、图像视频编辑器、文本编辑器等。一旦用户允许了 Web 应用对本地文件的访问，此 API 可以使 Web 应用直接对用户设备上的文件和文件夹进行读与保存修改。除了读写文件，文件系统访问 API 还提供了打开目录以及列举目录内容的能力。



如果你曾经进行过文件读写，那你会很熟悉我下面分享的内容。不过我还是鼓励你读一下这篇文章，因为不是所有的系统都是类似的。

**（原作者注：我们庆祝倾注了很多想法到了文件系统访问 API 的设计与实现中，以确保人们能够轻松的管理他们的文件。查看 [Secruity and permissions] 部分来获取更多信息。)**

## 文章进度（原文中已经全部完成）

前往 https://web.dev/file-system-access/#status 查看

## 使用文件系统访问 API

为了展示文件系统访问 API 的能力与好用，我写了一个单文件[文本编辑器](https://googlechromelabs.github.io/text-editor/)。它可以让你打开一个文本文件，编辑它，将修改保存回本地磁盘，或者新建一个文件然后保存修改到本地磁盘。这并不是什么精致（fancy）的东西，但它足以帮你理解概念。

### 试试看

看看文件系统访问 API 在[文本编辑器](https://googlechromelabs.github.io/text-editor/) demo 中的表现。

### 从本地文件系统读取一个文件

我想要做的第一件事是让用户选择一个文件，然后打开并从磁盘上读取这个文件。

#### 让用户选择一个文件进行读取

文件系统访问 API 中的入口点（entry point）是 [`window.showOpenFilePicker()`](https://wicg.github.io/file-system-access/#api-showopenfilepicker。当调用它时，浏览器会弹出一个文件选择对话框，并让用户选择一个文件。

未完待续。。。

To be continued ...

つづく。。。