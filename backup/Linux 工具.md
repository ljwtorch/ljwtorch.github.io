# Linux工具

## psmisc

psmisc 是一个 Linux 工具集，它提供了一些管理和监视进程的实用工具。它的名字源自 "process status miscellaneous"。psmisc 工具集包含了一些常用的进程管理工具，例如 `ps`、`killall`、`fuser` 等等。

以下是一些示例用法：

1. **ps**: 显示当前系统中的进程信息；

   ```shell
   bashCopy code
   ps aux               # 显示当前所有进程的详细信息
   ps -ef               # 显示所有进程的详细信息
   ps -u username       # 显示特定用户的进程信息
   ps -p PID            # 显示特定进程ID的信息
   ```

2. **killall**: 根据进程名字杀死进程；

   ```shell
   bashCopy code
   killall firefox      # 杀死所有名为 firefox 的进程
   killall -u username  # 杀死特定用户的所有进程
   ```

3. **fuser**: 显示正在使用指定文件或文件系统的进程信息；

   ```shell
   bashCopy code
   fuser -v /dev/sda1   # 显示使用 /dev/sda1 的进程信息
   fuser -k /mnt        # 杀死使用 /mnt 目录的进程
   ```

4. **pstree**: 以树形结构显示进程关系；

   ```shell
   bashCopy code
   pstree               # 显示当前系统中所有进程的树形结构
   pstree -p            # 显示进程树以及对应的进程ID
   ```

5. **peekfd**: 显示文件描述符的信息；

   ```shell
   bashCopy code
   peekfd -p PID        # 显示特定进程的文件描述符信息
   ```

6. `psloginfo`: 分析系统日志文件，并按时间顺序显示系统启动后产生的进程信息；

7. `nice/renice`: 改变进程的优先级（nice值），从而影响其CPU调度优先级；

## rsync

### 1.安装

```shell
sudo apt install rsync
#服务器双方都要安装！
```

### 2.基本用法

#### 2.1 -r 参数

本机使用 rsync 命令时，可以作为`cp`和`mv`命令的替代方法，将源目录同步到目标目录。

```shell
rsync -r source destination
```

上面命令中，`-r`表示递归，即包含子目录。注意，`-r`是必须的，否则 rsync 运行不会成功。`source`目录表示源目录，`destination`表示目标目录。

如果有多个文件或目录需要同步，可以写成下面这样。

```shell
rsync -r source1 source2 destination
```

上面命令中，`source1`、`source2`都会被同步到`destination`目录。

#### 2.2 -a 参数

`-a`参数可以替代`-r`，除了可以递归同步以外，还可以同步元信息（比如修改时间、权限等）。由于 rsync 默认使用文件大小和修改时间决定文件是否需要更新，所以`-a`比`-r`更有用。下面的用法才是常见的写法。

```shell
rsync -r source1 source2 destination
```

目标目录`destination`如果不存在，rsync 会自动创建。执行上面的命令后，源目录`source`被完整地复制到了目标目录`destination`下面，即形成了`destination/source`的目录结构。

如果只想同步源目录`source`里面的内容到目标目录`destination`，则需要在源目录后面加上斜杠。

```shell
rsync -a source/ destination
```

上面命令执行后，`source`目录里面的内容，就都被复制到了`destination`目录里面，并不会在`destination`下面创建一个`source`子目录。

#### 2.3 -n 参数

如果不确定 rsync 执行后会产生什么结果，可以先用`-n`或`--dry-run`参数模拟执行的结果。

```shell
rsync -anv source/ destination
```

上面命令中，`-n`参数模拟命令执行的结果，并不真的执行命令。`-v`参数则是将结果输出到终端，这样就可以看到哪些内容会被同步。

#### 2.4 --delete 参数

默认情况下，rsync 只确保源目录的所有内容（明确排除的文件除外）都复制到目标目录。它不会使两个目录保持相同，并且不会删除文件。如果要使得目标目录成为源目录的镜像副本，则必须使用`--delete`参数，这将删除只存在于目标目录、不存在于源目录的文件。

```shell
rsync -av --delete source/ destination
```

上面命令中，`--delete`参数会使得`destination`成为`source`的一个镜像。

### 3.排除文件

#### 3.1 --exclude

同步时排除某些文件或目录：

```shell
rsync -av --exclude='*.txt' source/ destination
# 或者
rsync -av --exclude '*.txt' source/ destination
```

*注：rsync会同步以点开头的文件，如果要排除隐藏文件，可以`--exclude=".*"`*

```shell
#如果要排除某个目录里面所有的文件，但不排除目录本身：
rsync -av --exclude 'dir1/*' source/ destination

#多个排除模式可以用多个--exclude参数
rsync -av --exclude 'file1.txt' --exclude 'dir1/*' source/ destination

#多个排除模式可以使用Bash大括号的扩展功能，可以只使用一个--exclude参数
rsync -av --exclude={'file1.txt','dir1/*'} source/ destination

#如果要排除的文件过多，可以写入到一个文件中，每个模式一行，使用--exclude--from参数指定这个文件
rsync -av --exclude-from='exclude-file.txt' source/ destination
```

#### 3.2 --include

用来指定必须同步的文件模式，往往与--exclude结合使用

```shell
#下面命令指定同步时，排除所有文件，但是会包括txt文件
rsync -av --include="*.txt" --exclude='*' source/ destination
```

### 4.远程同步

#### 4.1 SSH同步

```shell
#可以将本地内容同步到远程服务器
rsync -av source/ username@remote_host:destination
#也可以将远程内容同步到本地
rsync -av username@remote_host:source/ destination

#如果ssh命令有附加的参数，必须使用-e参数指定要执行的ssh命令
rsync -av -e 'ssh -p 10000' source/ user@remote_host:/destination
```

#### 4.2 rsync协议

除了SSH，如果另一台服务器安装并运行了rsync守护程序，可以使用rsync://协议，端口873，进行传输：

```shell
#写法是服务器与目标目录之间使用双冒号分隔::	下面指令中的module不是实际路径名，而是rsync守护程序指定的一个资源名，由管理员分配
rsync -av source/ 192.168.122.32::module/destination

#列出rsync守护程序分配的所有module列表：
rsync rsync://192.168.122.32

#除了使用双冒号，也可以使用rsync://协议指定地址
rsync -av source/ rsync://192.168.122.32/module/destination
```

### 5.增量备份

rsync支持增量备份，除了源目录与目标目录直接比较，rsync 还支持使用基准目录，即将源目录与基准目录之间变动的部分，同步到目标目录。

大概流程是：第一次同步是全量备份，所有文件在基准目录里同步一份，以后每一次同步都是增量备份，只同步源目录与基准目录之间有变动的部分，将这部分保存在一个新的目标目录，这个新的目标目录之中，也是包含所有文件，但实际上，只有那些变动过的文件是存在于该目录，其他没有变动的文件都是指向基准目录文件的硬链接。

```shell
#--link-dest参数用来指定同步时的基准目录
rsync -a --delete --link-dest /compare/path /source/path /target/path
```

增量备份脚本示例：

```shell
#!/bin/bash
# A script to perform incremental backups using rsync
set -o errexit	#发生错误时立即退出shell
set -o nounset	#尝试使用未设置的变量时引发错误
set -o pipefail	#管道命令中，任一命令失败则整个管道命令标记为失败

#readonly关键字可以声明一个变量为只读，不能被修改或重新赋值
readonly SOURCE_DIR="${HOME}"
readonly BACKUP_DIR="/mnt/data/backups"
readonly DATETIME="$(date '+%Y-%m-%d_%H:%M:%S')"
readonly BACKUP_PATH="${BACKUP_DIR}/${DATETIME}"
readonly LATEST_LINK="${BACKUP_DIR}/latest"

mkdir -p "${BACKUP_DIR}"
#指定源目录的路径——指定一个备份目录作为硬链接参考目录——排除目录——指定备份文件的路径
rsync -av --delete "${SOURCE_DIR}/" --link-dest "${LATEST_LINK}" --exclude=".cache" "${BACKUP_PATH}"

#删除之前最新备份的符号链接
rm -rf "${LATEST_LINK}"
#创建一个指向当前备份的符号链接，以便标记为最新备份
ln -s "${BACKUP_PATH}" "${LATEST_LINK}"
```

### 6.配置

`-a`、`--archive`参数表示存档模式，保存所有的元数据，比如修改时间（modification time）、权限、所有者等，并且软链接也会同步过去。

`--append`参数指定文件接着上次中断的地方，继续传输。

`--append-verify`参数跟`--append`参数类似，但会对传输完成后的文件进行一次校验。如果校验失败，将重新发送整个文件。

`-b`、`--backup`参数指定在删除或更新目标目录已经存在的文件时，将该文件更名后进行备份，默认行为是删除。更名规则是添加由`--suffix`参数指定的文件后缀名，默认是`~`。

`--backup-dir`参数指定文件备份时存放的目录，比如`--backup-dir=/path/to/backups`。

`--bwlimit`参数指定带宽限制，默认单位是 KB/s，比如`--bwlimit=100`。

`-c`、`--checksum`参数改变`rsync`的校验方式。默认情况下，rsync 只检查文件的大小和最后修改日期是否发生变化，如果发生变化，就重新传输；使用这个参数以后，则通过判断文件内容的校验和，决定是否重新传输。

`--delete`参数删除只存在于目标目录、不存在于源目标的文件，即保证目标目录是源目标的镜像。

`-e`参数指定使用 SSH 协议传输数据。

`--exclude`参数指定排除不进行同步的文件，比如`--exclude="*.iso"`。

`--exclude-from`参数指定一个本地文件，里面是需要排除的文件模式，每个模式一行。

`--existing`、`--ignore-non-existing`参数表示不同步目标目录中不存在的文件和目录。

`-h`参数表示以人类可读的格式输出。

`-h`、`--help`参数返回帮助信息。

`-i`参数表示输出源目录与目标目录之间文件差异的详细情况。

`--ignore-existing`参数表示只要该文件在目标目录中已经存在，就跳过去，不再同步这些文件。

`--include`参数指定同步时要包括的文件，一般与`--exclude`结合使用。

`--link-dest`参数指定增量备份的基准目录。

`-m`参数指定不同步空目录。

`--max-size`参数设置传输的最大文件的大小限制，比如不超过200KB（`--max-size='200k'`）。

`--min-size`参数设置传输的最小文件的大小限制，比如不小于10KB（`--min-size=10k`）。

`-n`参数或`--dry-run`参数模拟将要执行的操作，而并不真的执行。配合`-v`参数使用，可以看到哪些内容会被同步过去。

`-P`参数是`--progress`和`--partial`这两个参数的结合。

`--partial`参数允许恢复中断的传输。不使用该参数时，`rsync`会删除传输到一半被打断的文件；使用该参数后，传输到一半的文件也会同步到目标目录，下次同步时再恢复中断的传输。一般需要与`--append`或`--append-verify`配合使用。

`--partial-dir`参数指定将传输到一半的文件保存到一个临时目录，比如`--partial-dir=.rsync-partial`。一般需要与`--append`或`--append-verify`配合使用。

`--progress`参数表示显示进展。

`-r`参数表示递归，即包含子目录。

`--remove-source-files`参数表示传输成功后，删除发送方的文件。

`--size-only`参数表示只同步大小有变化的文件，不考虑文件修改时间的差异。

`--suffix`参数指定文件名备份时，对文件名添加的后缀，默认是`~`。

`-u`、`--update`参数表示同步时跳过目标目录中修改时间更新的文件，即不同步这些有更新的时间戳的文件。

`-v`参数表示输出细节。`-vv`表示输出更详细的信息，`-vvv`表示输出最详细的信息。

`--version`参数返回 rsync 的版本。

`-z`参数指定同步时压缩数据。



## screen

### 1.安装

`apt install screen`

### 2.基础命令

```shell
# 终端列表
screen -ls

# 新建终端，一般用-R创建虚拟终端
# 使用-R创建，如果之前有创建唯一一个同名的screen，则直接进入之前创建的screen
# 使用-S创建和直接输入screen创建的虚拟终端，不会检录之前创建的screen（也就是会创建同名的screen)
screen -R test

# 返回主终端
ctrl+a d

# 回到终端 -R -r 都可以
screen -r [pid/name]

# 退出终端
exit

```

## trzsz

### 1.安装&使用

https://github.com/trzsz/trzsz-go

