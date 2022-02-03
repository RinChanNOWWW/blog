---
title: 关于 LevelDB 的一些问题（个人向）
date: 2022-02-03 19:28:51
tags:
    - 存储
    - 数据库
    - LSM-Tree
    - LevelDB
categories:
    - 源码笔记
---

记录一下学习 LevelDB 遇到的一些问题。

<!-- more -->

## 为什么每次重新打开 LevelDB 都会生成新的文件

如果一开始就不存在 DB 相关文件，自然会创建新的，这里主要看一下已经存在旧的文件时的情况。

### MANIFEST 文件

先来看 *MANIFEST* 这个文件为什么每次都会生成一个新的。在方法 `DB::Open` 中，会调用方法 `DBImpl::Recover(VersionEdit* edit, bool* save_manifest)` 将 DB 信息恢复到内存中，其中的参数 `save_manifest` 就代表这是否需要将内存中的元信息写入到新的 *MANIFEST* 文件中。

首先会进入到方法 `VersionSet::Recover(bool* save_manifest)` (*db/version_set.cc*) 中。这个方法会读取老的 *MANIFEST* 文件，恢复整个 DB 的元信息（比如 comparator name、log number、last seq number、new file entry 等）。在方法的最后，会调用 `VersionSet::ReuseManifest` 来判读是否需要重新生成新的 *MANIFEST* 文件：

- 如果 `Options` 中没有设置 `reuse_logs` 这个选项，则默认会生成新的 *MANIFEST* 文件 (`return false`)。
- 否则会进行一系列的信息检测，如果都通过，则会不会生成新的 *MANIFEST* 文件 (`return true`)。这些检测包括：
    - 当前的 DB 文件是否都合法。
    - 当前的 *MANIFEST* 是否是 appendable 的。

回到 `DB::Oepn` 后若 `save_manifest` 为 `true`，则通过 `VersionSet::LogAndApply` 写入新的 *MANIFEST* 文件。然后再调用 `DBImpl::RemoveObsoleteFiles` 来删掉旧的 *MANIFEST* 文件。

涉及 *MANIFEST* 文件的伪代码大致如下：

```cpp
// db/db_impl.cc

Status DB::Open(const Options& options, const std::string& dbname, DB** dbptr) {
    // ...
    Status s = impl->Recover(&edit, &save_manifest);
    // ...
    if (s.ok() && save_manifest) {
        // ...
        s = impl->versions_->LogAndApply(&edit, &mutex_);
    }
    if (s.ok()) {
        // ...
        impl->RemoveObsoleteFiles();
        // ...
    }
    // ...
}

Status DBImpl::Recover(VersionEdit* edit, bool* save_manifest) {
    // ...
    s = versions_->Recover(save_manifest);
    // ...
}

// db/version_set.cc

Status VersionSet::Recover(bool* save_manifest) {
    Status s = RecoverFromManifest();
    // ...
    if (s.ok()) {
        if (ReuseManifest(dscname, current)) {
            // No need to save new manifest
        } else {
            *save_manifest = true;
        }
    }
    // ...
}

bool VersionSet::ReuseManifest(const std::string& dscname,
                               const std::string& dscbase) {
    if (!options_->reuse_logs) {
        return false;
    }
    // ...
    if (!ParseFileName(dscbase, &manifest_number, &manifest_type) ||
        manifest_type != kDescriptorFile ||
        !env_->GetFileSize(dscname, &manifest_size).ok() ||
        // Make new compacted MANIFEST if old one is too big
        manifest_size >= TargetFileSize(options_)) {
        return false;
    }
    Status r = env_->NewAppendableFile(dscname, &descriptor_file_);
    if (!r.ok()) {
        return false;
    }
    // ...
    return true;
}

```

### .log 文件

每次重启 DB 都会生成新的 *.log* 文件，但是创建时机有所不同。

最简单的是，旧的 *.log* 文件中没有记录，所以在之前的 `DBImpl::Recover` 过程中不会简历 `MemTable`，也就是 `impl_->mem_` 为 `nullptr`，程序会以这个依据在 `DB::Open` 的逻辑中为 DB 创建新的 *.log* 文件并在内存中创建新的 `Memtable`。

如果旧的 *.log* 文件有记录，则会在 `DBImpl::Recover` 中调用的 `DBImpl::RecoverLogFile` 中读取记录，并在这个时候建立 `MemTable`，并将 log 应用到 `MemTable` 中，并立刻持久化到 *.ldb* 文件中（minor compaction）。这里并不会将创建的 `MemTable` 保存在内存中（也就是不会赋值给 `impl_`，**因为需要将已经落盘的 `MemTable` 从内存中丢弃**），等逻辑回到 `DB::Open` 中后，会和上面所说的 *.log* 中没有记录一样，进入 `impl->mem_ == nullptr` 的逻辑，统一创建新的 *.log* 文件与 `MemTable`。

顺带一提的是，其实可以设置 `Options` 中的 `reuse_logs` 选项来选择复用旧的 *.log* 文件，但是前提是 *.log* 是 appendable 的，并且没有经历过 compaction（如果 `MemTable` 中的数据量过大，会在判断 `reuse_logs` 选项之前进行 minor compaction，将 `MemTable` 落盘，**需要将已经落盘的 `MemTable` 从内存中丢弃**，这之后就会进入上面所说的创建新 log 的逻辑，所以就没法复用旧 log 了）。如果没有经历过 compaction，则可以直接保留 `MemTable` 到 `impl_->mem` 中，跳过之后创建新 log 和新 `MemTable` 的逻辑。

最后与 *MANIFEST* 文件同理，会通过 `DBImpl::RemoveObsoleteFiles` 清理掉旧的 *.log* 文件。这里需要注意的是，一旦 Recover 了 log 并 dump 到了 SST 之后，旧的 log 就作废不能再使用了，必须将其删除，如果在上述 `reuse_logs` 的逻辑中继续使用经历过 minor compaction 的 log，则会导致再次重启时重复写入已经 dump 过了的大量数据，影响性能。

涉及 *.log* 文件的恢复流程大致伪代码如下:

```cpp
// db/db_impl.cc

Status DB::Open(const Options& options, const std::string& dbname, DB** dbptr) {
    // ...
    Status s = impl->Recover(&edit, &save_manifest);
    if (s.ok() && impl->mem_ == nullptr) {
        s = NewLogFile();
        if (s.ok()) {
            edit.SetLogNumber(new_log_number);
            impl->logfile_ = lfile;
            impl->logfile_number_ = new_log_number;
            impl_mem_ = NewMemTable();
        }
    }
    // ...
    if (s.ok()) {
        impl->RemoveObsoleteFiles();
        // ...
    }
    // ...
}

Status DBImpl::Recover(VersionEdit* edit, bool* save_manifest) {
    // ...
    std::vector<uint64_t> logs;
    GetLogs(&logs);
    std::sort(logs.begin(), logs.end());
    for (log in logs) {
        s = RecoverLogFile(
            logs[i], 
            (i == logs.size() - 1), 
            save_manifest, 
            edit,
            &max_sequence);
        // ...
    }
    // ...
}

Status DBImpl::RecoverLogFile(uint64_t log_number, bool last_log,
                              bool* save_manifest, VersionEdit* edit,
                              SequenceNumber* max_sequence) {
    // ...
    while (reader.ReadRecord(&record, &scratch) && status.ok()) {
        // ...
        if (mem == nullptr) {
            mem = NewMemTable();
        }
        status = InsertInto(&batch, mem);
        // ...
        if (mem->ApproximateMemoryUsage() > options_.write_buffer_size) {
            compactions++;
            *save_manifest = true;
            status = WriteLevel0Table(mem, edit, nullptr);
            // ...
        }
    }
    // ...
    if (status.ok() && options_.reuse_logs && last_log && compactions == 0) {
        if (NewAppendableFile(logfilename, &logfile_)) {
            // ...
            if (mem != nullptr) {
                mem_ = mem;
                mem = nullptr;
            } else {
                mem_ = NewMemTable();
            }
            // ...
        }
    }
    if (mem != nullptr) {
        // ...
        if (status.ok()) {
            *save_manifest = true;
            status = WriteLevel0Table(mem, edit, nullptr);
        }
        // ...
    }
    // ...
}
```

### .ldb 文件

如上文所说，在重新打开 DB 后，如果旧的 *.log* 文件中存在写入记录，则会根据 log 建立 `MemTable`，并调用 `DBImpl::WriteLevel0Table` 将刚建立 `MemTable` 写入（或称 dump）到 *.ldb* 文件，并将 `MemTable` 删除。值得注意的是，这里虽然方法名字叫 `WriteLevel0Table`，但是其实不一定是写到 Level-0，LevelDB 可能会按照一定策略提前将 SST 写到更深层的 Level 中，这部分留到之后再看了。如果 log 中没有相关记录则不会创建新的 *.ldb* 文件。这个 *.ldb* 文件也就是 SST，是对某一时刻的 `MemTable` 的 dump，也就是说一般情况下都是只读的。

在 `DB::Open` 的最后，还会调用 `DBImpl::MaybeScheduleCompaction` 在后台开启负责 compaction 的线程，对满足条件的 *.ldb* 文件进行 major compaction，这部分也留到之后再看。