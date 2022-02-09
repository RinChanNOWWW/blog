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

最后更新：2022-02-09。

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

最简单的是，旧的 *.log* 文件中没有记录，所以在之前的 `DBImpl::Recover` 过程中不会建立 `MemTable`，也就是 `impl_->mem_` 为 `nullptr`，程序会以这个依据在 `DB::Open` 的逻辑中为 DB 创建新的 *.log* 文件并在内存中创建新的 `Memtable`。

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
        s = NewLogFile(); // pseudo code
        if (s.ok()) {
            edit.SetLogNumber(new_log_number);
            impl->logfile_ = lfile;
            impl->logfile_number_ = new_log_number;
            impl_mem_ = NewMemTable(); // pseudo code
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
            mem = NewMemTable(); // pseudo code
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
                mem_ = NewMemTable(); // pseudo code
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

如上文所说，在重新打开 DB 后，如果旧的 *.log* 文件中存在写入记录，则会根据 log 建立 `MemTable`，并调用 `DBImpl::WriteLevel0Table` 将刚建立 `MemTable` 写入（或称 dump）为一个新的 *.ldb* 文件，并将 `MemTable` 删除。值得注意的是，这里虽然方法名字叫 `WriteLevel0Table`，但是其实不一定是写到 Level-0，LevelDB 可能会按照一定策略提前将 SST 写到更深层的 Level 中，这部分留到之后再看了。如果 log 中没有相关记录则不会创建新的 *.ldb* 文件。这个 *.ldb* 文件也就是 SST，是对某一时刻的 `MemTable` 的 dump，也就是说一般情况下都是只读的。

在 `DB::Open` 的最后，还会调用 `DBImpl::MaybeScheduleCompaction` 在后台开启负责 compaction 的线程，对满足条件的 *.ldb* 文件进行 major compaction，这部分也留到之后再看。

## DBImpl::Write 中为什么要调用 Sync()

如果 `WriteOptions` 中设置了 `sync` 选项，在写入 log 之后会立刻调用 `logfile_->Sync()`：

```cpp
status = log_->AddRecord(WriteBatchInternal::Contents(write_batch));
bool sync_error = false;
if (status.ok() && options.sync) {
    status = logfile_->Sync();
    if (!status.ok()) {
        sync_error = true;
    }
}
```

我的疑惑是 `AddRecord` 的实现中已经调用了 `write` 将内容写到了对应的文件描述符对应的文件，为什么还需要 `Sync`？这里其实是因为现代操作系统一般执行写入的时候都只是先把内容写到缓存中，极其一定大小的数据块后才统一 flush 到磁盘中，这样加快了写操作的速度。而调用 `Sync` 就相当于强制刷盘，进行一个同步的等待。拿实现了 POSIX 接口的环境来说（执行逻辑进入 *util/env_posix.cc*），最终会调用 `fcntl(fd, F_FULLFSYNC)` 或 `fdatasync(fd)` 或 `fsyn(fd)` ，这取决于当前系统的支持，其目的都是进行一个强制落盘的同步操作。所以说开启了 `sync` 这个选项可能使写操作变慢。如果 `Sync` 失败，则会记录一个 `bg_error_`（background error），使得之后所有的写操作都失败，源码中的注释写的是：

> The state of the log file is indeterminate: the log record we just added may or may not show up when the DB is re-opened. So we force the DB into a mode where all future writes fail.

我也不知道进入这个失败模式后会不会通过一些措施恢复，针对 `bg_error_` 搜索了一番发现并没有恢复的逻辑，可能进入这个模式之后就只能靠应用层重启 DB 了吧。

## DBImpl::Write 使怎么使用 writers 队列的

一开始这个 writers 队列，还有 writer 的 condition variable 还有最后的 while 循环 pop writers 队列的一系列操作看的我很懵逼，心想这到底有什么意义，看起来和加一把大锁串行执行没什么区别。理解这一段代码着重需要理解这两个东西：

- C++ 中 condition variable 的用法。
- `DBImpl::BuildBatchGroup(Writer** last_writer)` 这个方法的作用。

对于 condition variable 的实现，正好在 6.S081 中才学习到过，所以印象还算深刻。这里用到了 C++11 引入的 `condition_variable`，以及它的两个方法 `wait` 和 `notify_one`，其中 `wait` 的作用是让当前线程进入睡眠，指导有另一个线程调用了 `notify_one`（或 `notify_all`）唤醒它。这里有一个要点是 **`wait` 还接收一个 lock，因为它会在进入睡眠之前 release 这个 lock，并在被唤醒后重新 acquire 这个 lock**，如果一个 sleep 了的线程 hold 一个 lock，那不就一直死锁了嘛，至于为什么需要这样一个锁，那是因为 `condition_variable` 的操作往往通常出现在加锁的逻辑中（也就是需要对共享内存进行并发原子操作的场景），关于 `condition_variable` 更具体地细节可以查阅 https://en.cppreference.com/w/cpp/thread/condition_variable 。

对于 `DBImpl::BuildBatchGroup(Writer** last_writer)` 方法，不看的话还真会忽略掉一个最重要的东西，那就是在这个方法**会将当前 writers 队列中的所有 writer 构建为一个 `WriterBatch`，并将 `last_writer` 更新为最后一个 writer**（但是有个例外是，带了 `sync` 选项的不会和不带的一起）。

现在再来看着一段代码就很清晰了：

```cpp
Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates) {
    // ...
    MutexLock l(&mutex_);
    writers_.push_back(&w);
    while (!w.done && &w != writers_.front()) {
        w.cv.Wait();
    }
    if (w.done) {
        return w.status;
    }
    // ...
    if (status.ok() && updates != nullptr) { 
        WriteBatch* write_batch = BuildBatchGroup(&last_writer);
        //...
        {
            mutex_.Unlock();
            WriteLogAndMemTable(); // pseudo code
            mutex_.Lock();
        }
        // ...
    }
    while (true) {
        Writer* ready = writers_.front();
        writers_.pop_front();
        if (ready != &w) {
            ready->status = status;
            ready->done = true;
            ready->cv.Signal();
        }
        if (ready == last_writer) break;
    }

    // Notify new head of write queue
    if (!writers_.empty()) {
        writers_.front()->cv.Signal();
    }

    return status;
}
```
首先 `MutexLock l(&mutex_);` 这段代码会先锁定 `mutex_`（类似 `scoped_lock`），最开始由于锁的存在只会进入一个 writer，并且不会进行 wait。当这个 writer 构建 `WriteBatch` 完毕之后便可释放 lock（因为每一个 `WriteBatch` 的落盘是互不影响的，它们之前没有需要共享的数据，`MemTable` 也有自己的锁）。在 lock 释放后并执行写入的这段时间，其他等待的 writer 便可以进入之后的被 push 进 writers 队列并进入睡眠状态。前一个 `WriteBatch` 的写入时间越长便越能积累更多的 writer。当前一个 `WriteBatch` 执行完毕后，就会查看 writer 队列，并唤醒一个 writer。下一个 writer 便可以同时将当前队列中的所有 writer 构建为一个 `WriteBatch`，并将 `last_writer` 更新为最后一个 writer。之后的 while 循环便是将已经被统一操作的 writer pop 出队列，并结束它们 block 住的线程。

`DBImpl::Write` 的设计让我在数据库写操作的设计与 C++ 多线程编程上学到很多（~~太棒了，学到昏厥~~）。代码逻辑清晰，可读性强，注释完备，连我那么菜的人都能轻松读懂，LevelDB 的代码质量可见一斑。

## 何时触发后台 Compaction

直接查看 `DBImpl::MaybeScheduleCompaction` 这个方法的调用地点：

- `DB::Open` 完成之后会立刻进行一次。
- 在 `Get` 操作中，如果查找进入到了 SST 文件中，并将此文件的 `allowed_seek` 数量耗尽，并且没有其他正在 compact 的文件，便会将此文件设置为即将 compact 的文件并尝试启动后台 compaction 线程。
- 通过 `DBIter` 迭代时，会设定一个 `bytes_until_read_sampling_`，这个值是 0 ~ 1MB 中的一个正态分布随机值，如果迭代的数据量达到了这个上限，便会进行一次 sampling 操作，统计 SST 的读取统计信息，和上面的 `Get` 操作类似，如果某 SST 的 `allowed_seek` 耗尽，便会尝试启动后台 compaction 线程。
- `Put` 与 `Delete` 操作最终会调用 `DBImpl::Write`。调用 `DBImpl::Write` 时会首先调用 `DBImpl::MakeRoomForWrite` 为写操作预留空间。在此方法中，如果发现 mutable 的 `MemTable` 已满，则会将它变为 immutable，然后创建新的 mutable `MemTable`，并尝试启动后台 compaction 线程。

现在再来看看 `DBImpl::MaybeScheduleCompaction`。对于一个 DB，只会产生一个 compaction 线程。具体来说，后台会启动一个 `background_thread` 线程，作为调度器，它的作用是循环抽取后台任务队列 `background_work_queue_` 中的任务进行执行（如果队列为空会进入 wait 状态，有新的任务到来时再被唤醒），后续调用 `DBImpl::MaybeScheduleCompaction` 便会向任务队列推入一个 compaction 任务而不产生新的线程。这里我有一个疑惑是，源代码中是先进行唤醒，再向任务队列中放任务：

```cpp
// util/env_posix.cc

void PosixEnv::Schedule(
    void (*background_work_function)(void* background_work_arg),
    void* background_work_arg) {
    background_work_mutex_.Lock();
    // ...
    // If the queue is empty, the background thread may be waiting for work.
    if (background_work_queue_.empty()) {
        background_work_cv_.Signal();
    }
    background_work_queue_.emplace(background_work_function, background_work_arg);
    background_work_mutex_.Unlock();
}

```

是否先 `emplace` 再 `Signal` 更好，比如：

```cpp
void PosixEnv::Schedule(
    void (*background_work_function)(void* background_work_arg),
    void* background_work_arg) {
    // ...
    bool empty = background_work_queue_.empty();
    background_work_queue_.emplace(background_work_function, background_work_arg);
    if (empty) {
        background_work_cv_.Signal();
    }
    // ...
}
```

但是其实这里并没有问题，先看看 `background_thread` 执行的代码：

```cpp
void PosixEnv::BackgroundThreadMain() {
    while (true) {
        background_work_mutex_.Lock();

        // Wait until there is work to be done.
        while (background_work_queue_.empty()) {
            background_work_cv_.Wait();
        }
        assert(!background_work_queue_.empty());
        // ...
        background_work_mutex_.Unlock();
        // ...
    }
}
```

这里 `background_work_cv_` 条件变量中的的 mutex 就是 `background_work_mutex_`，就像上一节「[DBImpl::Write 使怎么使用 writers 队列的](#DBImpl-Write-使怎么使用-writers-队列的)」中所说，从 wait 中苏醒后会立刻给 mutex 重新调用 lock，所以这里如果 `background_thread` 在 `Schedule` 的途中被唤醒，它并不会直接继续向下执行，而是会等待任务被推入队列然后释放锁之后才会进入后面的逻辑。

compaction 的调用链为 `DBImpl::BGWork` -> `DBImpl::BackgroundCall` -> `DBImpl::BackgroundCompaction`。

