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

文件系统访问 API（[The File System Access API](https://wicg.github.io/file-system-access/)，之前又叫 Native File System API，更之前叫 Writeable Files API）使得开发者们能够构建能与用户本地设备文件交互的强大 Web 应用，比如 IDE、图像视频编辑器、文本编辑器等。一旦用户允许了 Web 应用访问本地文件的请求，此 API 便可以使 Web 应用直接对用户设备上的文件和文件夹进行读写修改。除了读写文件，文件系统访问 API 还提供了打开目录以及列举目录内容的能力。

如果你曾经操作过文件读写，那你会很熟悉我下面分享的内容。不过我还是鼓励你读一下这篇文章，因为不是所有的系统都是类似的。

**（原作者注：对于文件系统访问 API 的设计与实现我们进行了深思熟虑，以确保人们能够轻松的管理他们的文件。查看 [Secruity and permissions] 部分来获取更多信息。)**

## 文章进度（原文中已经全部完成）

前往 https://web.dev/file-system-access/#status 查看

## 使用文件系统访问 API

为了展示文件系统访问 API 的强大与好用，我写了一个单文件[文本编辑器](https://googlechromelabs.github.io/text-editor/)。它可以让你打开一个文本文件，编辑它，将修改保存回本地，或者新建一个文件然后保存修改到本地。这并不是什么精致（fancy）的东西，但它足以帮你理解概念。

### 试试看

看看文件系统访问 API 在[文本编辑器](https://googlechromelabs.github.io/text-editor/) demo 中的表现。

### 从本地文件系统读取一个文件

我想要做的第一件事是让用户选择一个文件，然后打开并从磁盘上读取这个文件。

#### 让用户选择一个文件进行读取

文件系统访问 API 中的入口点（entry point）是 [`window.showOpenFilePicker()`](https://wicg.github.io/file-system-access/#api-showopenfilepicker)。当调用它时，浏览器会弹出一个文件选择对话框，并让用户选择一个文件。用户选择完文件后，此 API 会返回一个文件句柄（handle）数组。`option` 参数可以让你影响文件选择器的行为，比如允许用户选择多个文件，目录或者其他文件类型。如果你没有指定 `option` 参数，选择器只让用户选择一个文件。这对于一个文本编辑器来说恰到好处。

和众多强大的 API 一样，必须在一个[安全上下文（secure context）](https://w3c.github.io/webappsec-secure-contexts/)中完成对 `showOpenFilePicker()` 的调用并且只能在一个 user gesture（详见 Chromium 对此的定义） 中被调用。

```javascript
let fileHandle;
butOpenFile.addEventListener('click', async () => {
  // Destructure the one-element array.
  [fileHandle] = await window.showOpenFilePicker();
  // Do something with the file handle.
});
```

一旦用户选择了一个文件，`showOpenFilePicker()` 会返回一个句柄数组，在这种情况下会返回一个只有一个 [`FileSystemFileHandle`](https://wicg.github.io/file-system-access/#api-filesystemfilehandle)  对象的数组，这个对象包含了需要和文件进行交互的属性和方法。

我们需要维护一个对此句柄的引用以便后续使用，我们需要用它来进行保存文件或者其他文件操作。

#### 从文件系统中读取一个文件

现在你有了一个文件的句柄，这下你就能够拿到这个文件的属性或者访问这个文件本身。现在，我会简单地读读取它的内容。调用 `handle.getFile` 并返回一个 [`File`](https://w3c.github.io/FileAPI/) 对象，它包含一个二进制文件（blob）。为了拿到二进制文件中的数据，调用[它的方法](https://developer.mozilla.org/en-US/docs/Web/API/Blob)（`slice()` , `stream()`, `text()`, `arrayBuffer()`）。

```javascript
const file = await fileHandle.getFile();
const contents = await file.text();
```

`FileSystemFileHandle.getFile()` 返回的 `File` 对象只有对应的底层文件没有后被更改时可读。如果底层文件已经被修改饿了，那此 `File` 对象便会变为不可读，这时候你得重新调用 `getFile()` 来获取新的 `File` 对象来读取被更改的数据。

#### 将上面的操作组合到一起

当用户点击打开（Open）按钮，浏览器会弹出文件选择器。当选中一个文件，这个应用会把读到的内容放入一个 `<textarea>` 中。

```javascript
let fileHandle;
butOpenFile.addEventListener('click', async () => {
  [fileHandle] = await window.showOpenFilePicker();
  const file = await fileHandle.getFile();
  const contents = await file.text();
  textArea.value = contents;
});
```

### 将文件写入本地文件系统

在这个文本编辑器中，有两种方式来保存文件：**保存（Save）**和**另存为（Save As）**。**保存**使用之前的文件句柄简单地将修改写回原文件中。但是**另存为**创建了一个新的文件，并需要一个新的文件句柄。

#### 创建一个新文件

为了保存一个文件，我们需要调用 `showSaveFilePicker()` 来让文件选择器变为“保存”模式，此模式下，文件选择器让用户选择一个文件来进行保存。在这个文件编辑器中，我想让它自动加上 `.txt` 的扩展名，所以我提供了一些额外的参数。

```javascript
async function getNewFileHandle() {
  const options = {
    types: [
      {
        description: 'Text Files',
        accept: {
          'text/plain': ['.txt'],
        },
      },
    ],
  };
  const handle = await window.showSaveFilePicker(options);
  return handle;
}
```

#### 保存修改到本地磁盘

你可以 [Github](https://github.com/GoogleChromeLabs/text-editor/) 上找到我这个[文本编辑器](https://googlechromelabs.github.io/text-editor/) demo 的所有代码。核心的文件系统交互部分在 `fs-helpers.js` 中。简单来说，整个过程如以下代码所示。我会一步一步地进行解释。

```javascript
async function writeFile(fileHandle, contents) {
  // Create a FileSystemWritableFileStream to write to.
  const writable = await fileHandle.createWritable();
  // Write the contents of the file to the stream.
  await writable.write(contents);
  // Close the file and write the contents to disk.
  await writable.close();
}
```

向磁盘中写入数据需要利用 [`FileSystemWritableFileStream`](https://wicg.github.io/file-system-access/#api-filesystemwritablefilestream) 对象，其本质是一个  [`WritableStream`](https://developer.mozilla.org/en-US/docs/Web/API/WritableStream)。调用 `createWritable()` 为文件句柄对象创建文件流（stream）。调用 `createWritable()` 后，浏览器会先向用户请求文件的写权限。如果你拒绝了此请求，`createWritable()` 会抛出异常 `DOMException`，你的应用也没办法写这个文件。在这个文件编辑器中，`saveFile()` 方法会处理这些 `DOMException` 异常。

你可以从文件编辑器中获取你要写的字符串（string）作为 `write()` 方法的参数传入。也可以直接获取 [BufferSource](https://developer.mozilla.org/en-US/docs/Web/API/BufferSource) 或者 [Blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob)。例如，你可以直接向文件传输数据流（pipe a stream）：

```javascript
async function writeURLToFile(fileHandle, url) {
  // Create a FileSystemWritableFileStream to write to.
  const writable = await fileHandle.createWritable();
  // Make an HTTP request for the contents.
  const response = await fetch(url);
  // Stream the response into the file.
  await response.body.pipeTo(writable);
  // pipeTo() closes the destination pipe by default, no need to close it.
}
```

你可以在文件流中使用 [`seek()`](https://wicg.github.io/file-system-access/#api-filesystemwritablefilestream-seek) 或者 [`truncate()`](https://wicg.github.io/file-system-access/#api-filesystemwritablefilestream-truncate) 方法来精确定位文件中的位置，或者改变文件的大小。

**（原作者提醒：直到文件流关闭前修改都不会写入磁盘，可以通过调用 `close()` 或者等文件流自动关闭）**

### 将文件句柄存入数据库

文件句柄是可以序列化的，这意味着你可以将它们存入数据库中，或者调用 `postMessage()` 在相同的域（the same top-level origin）中传递它们。

将文件句柄存入数据库意味着你可以存储状态，或者记录下用户在使用哪些文件。这让你可以拥有一个最近打开或编辑过的文件列表，或者可以提供打开最近使用的文件的功能等等。在这个文本编辑器中，我将用户最近打开的五个文件存了起来，让用户能够方便地重新选择这些文件。

在不同会话（session）中文件地访问权限是不能持续存在的，所以你应该使用 `queryPermission()` 来检验用户是否允许对某文件的访问。如果没有，使用 `requestPermission()` 来重新请求。

在上面这个文本编辑器中，我定义了一个 `verifyPermission()` 方法来检查用户是否已经授予了权限， 如果没有，就会请求。

```javascript
async function verifyPermission(fileHandle, readWrite) {
  const options = {};
  if (readWrite) {
    options.mode = 'readwrite';
  }
  // Check if permission was already granted. If so, return true.
  if ((await fileHandle.queryPermission(options)) === 'granted') {
    return true;
  }
  // Request permission. If the user grants permission, return true.
  if ((await fileHandle.requestPermission(options)) === 'granted') {
    return true;
  }
  // The user didn't grant permission, so return false.
  return false;
}
```

通过在读请求时申请写权限，我减少了请求权限的次数：打开一个文件时用户只用允许一次，便可以同时授予应用读写权限。

### 打开一个目录并列出其内容

要列出目录下的所有文件，需要调用 [`showDirectoryPicker()` ](https://wicg.github.io/file-system-access/#api-showdirectorypicker)。用户在我呢见选择器中选择一个目录，然后会返回一个 [`FileSystemDirectoryHandle`](https://wicg.github.io/file-system-access/#api-filesystemdirectoryhandle)，这个句柄对象能让你列举并访问目录中的文件。

```javascript
const butDir = document.getElementById('butDirectory');
butDir.addEventListener('click', async () => {
  const dirHandle = await window.showDirectoryPicker();
  for await (const entry of dirHandle.values()) {
    console.log(entry.kind, entry.name);
  }
});
```

### 新建或者目录下的访问文件和文件夹

通过目录的句柄，你可以通过使用 [`getFileHandle()`](https://wicg.github.io/file-system-access/#dom-filesystemdirectoryhandle-getfilehandle) 与 [`getDirectoryHandle()`](https://wicg.github.io/file-system-access/#dom-filesystemdirectoryhandle-getdirectoryhandle) 创建或者访问文件和文件夹。你可以通过传入一个额外的 `option` 对象，并带有一个布尔类型的 `create` 字段，来决定如果文件或文件夹不存在时是否创建一个新的。

```javascript
// In an existing directory, create a new directory named "My Documents".
const newDirectoryHandle = await existingDirectoryHandle.getDirectoryHandle('My Documents', {
  create: true,
});
// In this new directory, create a file named "My Notes.txt".
const newFileHandle = await newDirectoryHandle.getFileHandle('My Notes.txt', { create: true });
```

### 解析目录中文件的路径

当你正在处理目录下的文件或文件夹时，解析它们的路径会对你很有用。这个操作可以通过调用 [`resolve()`](https://wicg.github.io/file-system-access/#api-filesystemdirectoryhandle-resolve) 实现。被解析的文件可以是目录的直接子女或者间接子女。

```javascript
// Resolve the path of the previously created file called "My Notes.txt".
const path = await newDirectoryHandle.resolve(newFileHandle);
// `path` is now ["My Documents", "My Notes.txt"]
```

### 删除目录下的文件和文件夹

如果你获取了访问一个目录的权限，那你就能够使用 [`removeEntry()`](https://wicg.github.io/file-system-access/#dom-filesystemdirectoryhandle-removeentry) 来删除它下面的文件与文件夹。删除文件夹时，你可以选择递归删除其所有子文件及其包含的文件。

```javascript
// Delete a file.
await directoryHandle.removeEntry('Abandoned Masterplan.txt');
// Recursively delete a folder.
await directoryHandle.removeEntry('Old Stuff', { recursive: true });
```

### 集成拖放（Drag and drop）功能

[HTML Drag and Drop interfaces](https://developer.mozilla.org/en-US/docs/Web/API/HTML_Drag_and_Drop_API) 让 Web 应用可以接受直接将文件拖放到网页中。在拖放操作中，拖拽文件或目录项将分别关联到文件或目录项的入口（entry）。当拖拽文件时，`DataTransferItem.getAsFileSystemHandle()` 方法会返回一个包含 `FileSystemFileHandle` 对象的 promise 对象，当拖拽目录时，一个包含 `FileSystemDirectoryHandle` 对象的 promise 对象。下面的代码展示了这一过程。注意，无论是文件还是目录， Drag and Drop interface 中的  [`DataTransferItem.kind`](https://developer.mozilla.org/en-US/docs/Web/API/DataTransferItem/kind) 都是 `"file"` ，然而文件系统访问 API 中的  [`FileSystemHandle.kind`](https://wicg.github.io/file-system-access/#dom-filesystemhandle-kind)  会区分为 `"file"` 和 `"direcotry"`。

```javascript
elem.addEventListener('dragover', (e) => {
  // Prevent navigation.
  e.preventDefault();
});

elem.addEventListener('drop', async (e) => {
  // Prevent navigation.
  e.preventDefault();
  // Process all of the items.
  for (const item of e.dataTransfer.items) {
    // Careful: `kind` will be 'file' for both file
    // _and_ directory entries.
    if (item.kind === 'file') {
      const entry = await item.getAsFileSystemHandle();
      if (entry.kind === 'directory') {
        handleDirectoryEntry(entry);
      } else {
        handleFileEntry(entry);
      }
    }
  }
});
```



未完待续。。。

To be continued ...

つづく。。。