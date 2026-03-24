# pjdfstest 失败测试分析

## 核心结论

**几乎所有失败的测试都指向同一个根本问题：你的文件系统在修改目录内容时，没有正确更新父目录的 `mtime`（修改时间）和 `ctime`（状态变更时间）。**

另有一组 `rename` 测试失败与 Btrfs 的 link count 语义相关。

---

## 失败测试详细分析

### 问题一：父目录时间戳未更新（共 38 个失败测试）

POSIX 规范要求：当目录内容发生变化（创建/删除文件、创建硬链接等）时，该目录的 `st_mtime` 和 `st_ctime` 必须更新。

以下所有失败测试都在验证同一件事：**操作成功后，父目录的 mtime 和 ctime 是否比操作前更大。**

| 测试文件 | 失败测试号 | 测试的操作 | 验证内容 |
|---------|-----------|-----------|--------|
| **mkdir/00.t** | 33-34 | `mkdir` 创建子目录 | 父目录 mtime/ctime 应更新 |
| **mkfifo/00.t** | 33-34 | `mkfifo` 创建 FIFO | 父目录 mtime/ctime 应更新 |
| **mknod/00.t** | 33-34 | `mknod` 创建 FIFO 节点 | 父目录 mtime/ctime 应更新 |
| **mknod/11.t** | 12-13 | `mknod` 创建字符设备 | 父目录 mtime/ctime 应更新 |
| **mknod/11.t** | 25-26 | `mknod` 创建块设备 | 父目录 mtime/ctime 应更新 |
| **open/00.t** | 33-34 | `open(O_CREAT)` 创建新文件 | 父目录 mtime/ctime 应更新 |
| **symlink/00.t** | 11-12 | `symlink` 创建符号链接 | 父目录 mtime/ctime 应更新 |
| **link/00.t** | 135-136 | `link` 为 regular file 创建硬链接 | 父目录 ctime/mtime 应更新 |
| **link/00.t** | 142-143 | `link` 为 FIFO 创建硬链接 | 父目录 ctime/mtime 应更新 |
| **link/00.t** | 149-150 | `link` 为 block device 创建硬链接 | 父目录 ctime/mtime 应更新 |
| **link/00.t** | 156-157 | `link` 为 char device 创建硬链接 | 父目录 ctime/mtime 应更新 |
| **link/00.t** | 163-164 | `link` 为 socket 创建硬链接 | 父目录 ctime/mtime 应更新 |
| **unlink/00.t** | 74-75 | `unlink` 删除 regular file | 父目录 mtime/ctime 应更新 |
| **unlink/00.t** | 80-81 | `unlink` 删除 FIFO | 父目录 mtime/ctime 应更新 |
| **unlink/00.t** | 86-87 | `unlink` 删除 block device | 父目录 mtime/ctime 应更新 |
| **unlink/00.t** | 92-93 | `unlink` 删除 char device | 父目录 mtime/ctime 应更新 |
| **unlink/00.t** | 98-99 | `unlink` 删除 socket | 父目录 mtime/ctime 应更新 |
| **unlink/00.t** | 104-105 | `unlink` 删除 symlink | 父目录 mtime/ctime 应更新 |
| **rmdir/00.t** | 8-9 | `rmdir` 删除子目录 | 父目录 mtime/ctime 应更新 |

**测试逻辑（所有测试相同模式）：**
```sh
time=`stat . ctime`      # 记录父目录当前 ctime
sleep 1                   # 等待 1 秒确保时间差
<执行操作>                # 例如 mkdir, unlink, link 等
mtime=`stat . mtime`     # 获取操作后的 mtime
test_check $time -lt $mtime   # 断言: 操作后 mtime > 操作前时间 (失败!)
ctime=`stat . ctime`     # 获取操作后的 ctime
test_check $time -lt $ctime   # 断言: 操作后 ctime > 操作前时间 (失败!)
```

**修复建议：** 在文件系统实现中，所有修改目录条目（dentry）的操作（`create`、`mkdir`、`mknod`、`link`、`unlink`、`rmdir`、`symlink`、`rename`）执行成功后，需要调用类似 `inode_update_time(dir, S_MTIME | S_CTIME)` 的函数来更新父目录 inode 的 mtime 和 ctime。

---

### 问题二：Btrfs 目录 link count 语义（2 个失败测试）

| 测试文件 | 失败测试号 | 验证内容 |
|---------|-----------|--------|
| **rename/24.t** | 4 | `src_parent` 的 nlink 应为 3（含子目录的 `..`） |
| **rename/24.t** | 9 | rename 后 `dst_parent` 的 nlink 应为 3 |

这些测试已标记为 `todo Linux "Btrfs uses CoW; link count semantics differ from POSIX."`，说明你的文件系统运行在 Btrfs 上或使用了类似的 CoW 语义。Btrfs 不维护传统的目录 nlink 计数（子目录的 `..` 不增加父目录的 link count），这是已知行为差异。

---

### 附：chown/00.t TODO passed 测试

chown/00.t 有 19 个标记为 `todo` 的测试意外通过了。这些测试涉及：
- 非 root 用户 chown 时目录的 SUID/SGID 位清除行为
- 当 owner 和 group 都为 -1 时 ctime 是否更新

这些是 Linux 与 POSIX 规范的已知行为差异，不是你的文件系统的问题。

---

## 总结

| 问题类别 | 影响的测试文件数 | 失败测试数 | 严重程度 |
|---------|---------------|-----------|--------|
| 父目录 mtime/ctime 未更新 | 9 个文件 | 38 个 | **高** — 核心 POSIX 合规问题 |
| Btrfs nlink 语义 | 1 个文件 | 2 个 | 低 — 已知平台差异 |
| **合计** | **10 个文件** | **40 个** | |

**最关键的修复点：确保所有修改目录内容的系统调用在成功完成后，更新该目录 inode 的 `mtime` 和 `ctime` 为当前时间。**
