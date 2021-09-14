---
title: ruborute 开发日志
date: 2021-08-29 20:01:59
tags:
  - Rust
  - 个人项目
categories:	
  - 编程开发
---

作为一个 SDVX 玩家，有一天在水群的时候看到了一位群友发了一下自己做的统计得分数据的脚本，于是催生了自己做一个查分器的想法。于是就有了 [ruborute](https://github.com/RinChanNOWWW/ruborute) 这个项目。它的读音为 "Are you 暴龍天(ぼるて, borute)?"。我打算用这个文章来记录我在开发过程中思路。

<!-- more -->

## 2021-9-14

### 实现 Tab 补全与历史提示

 [rustyline](https://github.com/kkawakam/rustyline) 提供了 `Completer` 和 `Hinter` 这两个 trait 可以用来实现这两个功能。对于历史提示（就像 zsh 的 auto-suggestion 插件那样），只需要接入 rustyline 提供的 `HistoryHinter` 即可，`Completer` 则需要自己实现。

```rust
/// To be called for tab-completion.
pub trait Completer {
    /// Takes the currently edited `line` with the cursor `pos`ition and
    /// returns the start position and the completion candidates for the
    /// partial word to be completed.
    ///
    /// ("ls /usr/loc", 11) => Ok((3, vec!["/usr/local/"]))
    fn complete(
        &self,
        line: &str,
        pos: usize,
        ctx: &Context<'_>,
    ) -> Result<(usize, Vec<Self::Candidate>)> {
        let _ = (line, pos, ctx);
        Ok((0, Vec::with_capacity(0)))
    }
}
```

只需要实现这个 complete 方法即可。这个方法参数是当前的输入，当前输入字符串的位置以及一个上下文（不用管），返回是要替换的字符串开始的位置以及候选项列表。如果候选项只有一个，它会直接不全，否则会显示所有候选项（需要开启 `CompletionType::List` 模式）。

实现这个主要参考了 rustyline 官方给的例子 https://github.com/kkawakam/rustyline/tree/master/examples （主要是那个 example.rs）。

## 2021-9-3

### 实现数据统计并发布 v0.1.1 版本

这几天为 ruborute 实现了一个 `count` 指令，用途是统计每一个 level 的 S、AAA+、AAA、PUC、PUC、HC、NC 的数量，以及这个 level 的总歌曲数。写这个的时候唯一的感想就是，Rust 的 match 竟然没有其他语言 switch case 中的 fallthrough 语义。然后统计 level 数量是利用迭代器 trait 中的 `filter().count()` 来计数，感觉可以在这个基础上做一个类似于 Python 中 pandas 那样的库啊（可能已经有了）。

之后想把 ruborute 上传到 crates.io 以及类似于 Chocolatey 这样的 Windows 包管理平台上，然后上传个 docs.rs 啥的，那还得完善一下注释才行。。。瞬间感受到开源工作者们的伟大，要做出一个高可用的开源软件出来要做的事情也太多了。

## 2021-8-31

### 发布 v0.1.0 版本

今天为 ruborute 打了 v0.1.0 的 tag。并创建了 Github Actions 实现持续集成（CD）。当发布 tag 时会自动发布 Releases 并上传此 tag 的 build target。一开始本来像给可执行文件 ruborute.exe 打个压缩包的，但是暂时还没搞明白在 Github Actions 的 Windows 环境中该怎么打压缩包，应该是要先安装打包软件，但是我没找到合适的。所以就直接上传可执行文件到 Releases 了。

## 2021-8-30

### 实现歌曲 Volforce 的计算

根据 BEMANIWIKI 上记载的[六代 VF 计算公式](http://bemaniwiki.com/index.php?SOUND%20VOLTEX%20EXCEED%20GEAR/VOLFORCE#calc)计算出每首歌的 VF，然后再求 VF 最高的 50 个记录的平均值。由于 Rust 的 `f64` 浮点数类型的比较实现起来比较繁琐，我将所有 VF 值都乘 1000 以整数方式记下来，方便比较与计算。不过我按照这个算下来的 VF 总比游戏里的高 0.01 左右，之后还得看看是怎么回事。

### 实现 Best 50 记录的查找

按照上面计算出来的 VF 值查找最高的 50 个即可。一开始本来想用一个最大堆，优先队列来实现，后来还是图简单直接排序取 Best 50 了。

### 修复可能出现重复记录的 BUG

今天发现第一天其实我错了，这个记录是有可能出现重复数据的，氧无服务器可能确实会在启动的时候会压缩一次记录，所以为了提高程序的鲁棒性，还是得在读取数据的时候加入判断重复的逻辑。

## 2021-8-28

### 实现通过歌曲名查找游玩记录

今天实现了通过歌曲名来查找游玩记录的功能。整体的思路是先到 `music_store` 通过音乐名找到对应的 `music_id`，然后再通过 `music_id` 到 `record_store` 中查找对应记录。在这里将 `record_store` 中通过 `music_id` 查找记录的方法改造成了批量读取，也就是参数是一系列 `music_id`，这样也易于功能的扩展。在歌名查找的问题上，我一开始想使用严格的正则匹配，但后来想没有几个人能记住完整的歌名，于是引入了模糊匹配（fuzzy matching），这里使用的轮子是 [rust-fuzzy-search](https://gitlab.com/EnricoCh/rust-fuzzy-search)，实现了模糊匹配算法，使用起来也比较简单。目前对于名称的搜索，模糊匹配的评分（score）定的大于 0.5，这个评分对于英文来说还不错，对于汉字可能要差点。然后目前还需要优化的一个点就是这个算法是大小写敏感的，之后只需要在 `filter` 方法的闭包函数统一一下大小写即可优化。

通过写这个功能，我发现 Rust 里像 `HashMap` 和 `Vec` 这样的容器的迭代器的方法是真的香啊，只需要简单地调用 `map`, `filter`, `flatten` 这些方法再 `collect` 一下就能完成把想要的数据从容器里洗出来，这是比 C++ STL 方便的一点。

### 在启动时组合完整信息

之前的 `record_store` 中只保存了 `savedata.db` 中的游玩记录信息，如果要输出完整信息，需要到 `music_store` 中查一下再组合成 `FullRecord` 输出。为了更加方便地编写业务代码，我决定在 ruborute 启动时，也就是读取数据文件时就组装出 `FullRecord` 然后存入 `record_store` 中。

## 2021-8-27

### 引入 music_db.xml 的音乐信息

今天的工作主要是接入游戏的音乐数据文件 `data/others/music_db.xml`，以支撑后续主要功能（查歌曲名、等级、VF等）的实现。不过要读这个 XML 文件可把我给难到了。XML Parser 的轮子自然不在少数，但是 `music_db.xml` 这个文件的编码是 `shift-jis` ，而 Rust 的默认读取格式是 UTF-8，而支持读取不同格式的 XML  库又异常的少。如果不做处理的话，一旦蹦到日文或者其他稀奇古怪的字符就会是乱码。所以急需一个支持读取非   UTF-8 编码的 XML 库。万幸的是，[quick-xml](https://github.com/tafia/quick-xml) 这个库的 encoding feature 支持不同编码。它通过 XML 文件一开始的 `xml` Event 中的 encoding 字段来判断整个文件的编码格式，然后用 [encoding_rs](https://github.com/hsivonen/encoding_rs) 这个库来解码，并转成 UTF-8 格式。而且更好的是，quick-xml 可以直接集成到 serde 中。虽然一开始我为了优化内存，并不想一口气读入整个 JSON 和 XML 文件，但是我并没有想到一个比较好的方法来使用 quick-xml + serde 流式读取 XML 文件，于是目前想的还是一次性载入所有数据到内存中（目前启动 ruborute 大概需要 1.2MB 内存）。实现了 `music_db.xml` 的读取之后，便可以顺利为歌曲数据输出歌曲名字和等级的信息了。

### 优化 Cmdline 模块

今天还未 `Cmd` 这个 trait 增加了 `name()`, `usage()`, `description()` 这三个方法。并增加了一个叫 `Cmdline` 的对象来保存实现了的 Cmd。优化了可扩展性，以后我加入命令只用专注于实现 `Cmd` ，然后再用 `add_command` 加入到 `Cmdline` 中即可，减少了后期工作量。

### 改造歌曲记录逻辑

今天发现了一首歌一个难度的记录只会记录一次，所以删除了一些判重的逻辑。然后将 `record_store` 中 `music_id` 到 `Record` 的记录改造成了 `Vec<Record>` ，因为会有同一首歌不同难度的记录。为了查询的便捷，就将一首歌存一块了。

## 2021-8-26

### 确定技术路线

最开始是为决定这个项目的技术路线。由于我用的游戏服务器采用的 JSON 来存储所有的游戏数据，所以需要自己撸一个存储引擎。一开始我就想到了曾经做过的 PingCAP Talent Plan 的项目 [simple-rust-kvs](https://github.com/RinChanNOWWW/simple-rust-kvs)，做一个 DB Server，然后通过网络请求来获取数据，再通过后端渲染一个 HTML 页面之类的。但是发现其实并没有这个必要，因为对于这样一个需求有这样的特点：

- 游玩数据不可能超过万的量级。
- 一切数据都是在本地，使用查分工具都是在本地，也没必要做成 server。
- 还要设计 HTML 页面太麻烦了。
- 没有高并发的必要。

于是我选择做一个 command-line 的工具，用户只需要指定数据所在路径，再结合各种 flag 和 arg 来完成需求。为了更高的可用性，我决定将 ruborute 设计成交互式的 cli 工具。

接下来就是编程语言的选择了。一开始我是想用 C++ 来写，但是转念一想，这个项目必定要用上各种第三方的轮子，比如 cli 参数的读取，数据的反序列化，以及展示数据等等。所以我选择了和 C++ 差不多 Rust，后者有强大的社区资源支撑，不怕找不到轮子，而且能帮我复健一下 Rust 这门基本不用的语言（为什么不用最熟悉的 Go 是因为实在不想写 Go 了）。

就这样我决定使用 Rust 来编写这个 cli 应用，它具有以下的特点：

- 以交互式命令行的形式运行，用户输入特定的指令进行交互。
- 单线程。
- 一次性载入数据到内存。
- 具有美观的展示内容。

### 确定要使用的开源库

对于命令行参数的实现，当然是选择了我们的老朋友 [clap](https://github.com/clap-rs/clap)，使用的是 3.0.0-beta.2 版本，这个版本可以使用 derive 来定义 struct。这里还有一个坑的地方就是，需要在 `Cargo.toml` 中指定 `calp = "=3.0.0-beta.2"`，不然 cargo 会下载 beta.4 的版本，然而这个版本会有 bug。

对于 JSON 的反序列化，当然也是我们的老朋友 [serde](https://github.com/serde-rs/serde) 和 [serde_json](https://github.com/serde-rs/json)。

对于交互式 io 的实现，我使用的是 [rustyline](https://github.com/kkawakam/rustyline)，这个库能够方便地读取数据以及控制字符，还能自己做一些美化（比如颜色啥的）。

然后就是数据结果的展示。我想的是通过表（Table）的形式打印出来，于是就选择了 [prettytable-rs](https://github.com/phsym/prettytable-rs) 这个看起来比较好用也好看的库。

对于命令功能的可扩展性实现，我定义了一个 `Cmd` 的 trait，它必须拥有 `do_cmd` 的功能，然后通过 Rust 的 dyn Trait 来通过用户输入在运行时指定相应 Cmd。

### 实现通过音乐 id 查找游玩记录

依靠以上开源组件，我实现了基本的从用户通过命令行启动 ruborute，再到输入交互式命令，再到打印出表结构数据的功能。
