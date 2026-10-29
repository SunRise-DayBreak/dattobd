# dattodb（整盘快照版）

dattobd 整盘快照版本：支持对带分区表的整盘（如 `/dev/sdb`）做只读一致性快照。

## 源码仓库

本目录即 dattobd 的 git 源码仓（fork 自 [datto/dattobd](https://github.com/datto/dattobd)），整盘快照定制改动在 `bada-beta.1` 分支：

- fork / origin：`https://github.com/SunRise-DayBreak/dattobd.git`
- upstream：`https://github.com/datto/dattobd.git`
- 定制分支：`bada-beta.1`（基于上游 `v0.12.1`，当前提交见 `git log`）
- 同步上游：`git fetch upstream && git merge --ff-only upstream/main`（定制改动保持在 `bada-beta.1`）

说明：
- 上游原始 README 保留为 `README.md`；本文件 `README-dattodb.md` 是整盘快照版的安装/编译/使用说明
- `prebuilt/` 为预编译产物，已在 `.gitignore` 中忽略，不入库

## 环境要求

- CentOS / RHEL 7.x x86_64，内核 3.10.x，root 权限
- 预编译模块适用于内核 `3.10.0-1160.71.1.el7.x86_64`；其它内核需从源码编译（gcc、make、kernel-devel-$(uname -r)）

## 安装部署

### 预编译模块（内核匹配时）

```bash
# 安装内核模块
cp prebuilt/dattobd.ko /lib/modules/$(uname -r)/extra/
depmod -a
modprobe dattobd
lsmod | grep dattobd

# 安装 dbdctl
cp prebuilt/dbdctl /usr/local/bin/dbdctl
chmod 755 /usr/local/bin/dbdctl
```

### 源码编译（内核不匹配时，在仓库根执行）

```bash
make -C src
make -C lib static CCFLAGS="-Wall -std=gnu99 -I$PWD/src"
make -C app static CCFLAGS="-Wall -std=gnu99 -I$PWD/src -I$PWD/lib" BASE_DIR="$PWD"

# 产物：src/dattobd.ko、app/dbdctl
cp src/dattobd.ko /lib/modules/$(uname -r)/extra/
depmod -a
modprobe dattobd
cp app/dbdctl /usr/local/bin/dbdctl
```

### 卸载

```bash
dbdctl destroy 0 2>/dev/null
rmmod dattobd
rm -f /lib/modules/$(uname -r)/extra/dattobd.ko /usr/local/bin/dbdctl
depmod -a
```

## 使用

```bash
# COW 文件必须放在另一块盘上
mkdir -p /backup-disk/cow

# 创建整盘快照（minor 0 → /dev/datto0）
dbdctl setup-snapshot /dev/sdb /backup-disk/cow/mirror 0

# 查看状态
cat /proc/datto-info

# 导出整盘镜像
dd if=/dev/datto0 of=/backup-disk/sdb.img bs=1M

# 或只读挂载快照分区（ext4 加 noload）
partx -a /dev/datto0
mount -o ro,noload /dev/datto0p1 /mnt/snap

# 销毁（COW 文件需手动删除）
umount /mnt/snap 2>/dev/null
dbdctl destroy 0
rm -f /backup-disk/cow/mirror
```

## 注意事项

- COW 必须放在另一块盘，放被快照整盘自己的分区会被拒绝。
- 快照前先 `sync`；需要强一致时用 `fsfreeze` 冻结各分区。
- 恢复到另一块盘：目标盘容量 ≥ 源盘，引导方式（BIOS/UEFI）需一致。
- xfs 只读挂载用 `nouuid,norecovery,ro`；COW 预分配默认整盘容量 10%，可用 `-f <MB>` 指定。
