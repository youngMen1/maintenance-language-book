# 第17课：磁盘数据恢复：rm -rf 误删数据，如何进行数据恢复

我们继续学习典型故障问题，主要是以“磁盘数据恢复”为主题的学习。 在工作中，我们知道一些操作命令危险性很高，如： rm -rf，它会造成数据的误删除。如果万一出现这样情况导致数据误删除时，我们应该如何对数据进行恢复呢？

## 删除数据的两种场景

通常有两种数据删除的场景是你需要清晰了解的。第 1 个是在执行 rm -rf 删除文件时，该文件正在被进程使用。第 2 个是这个文件并没有被其他进程所使用，而被误删除。本课时我将围绕这两种场景进行讲解并演示。

## 为什么数据可以恢复
既然我执行了 rm -rf 命令，不就是删除文件了吗，为什么又可以恢复数据呢？首先我来为你介绍一下其原由，对于第 1 种进程正在使用文件的场景，数据可以恢复是由因为 Linux 里，每个文件都有 2 个 link 计数器：i_count 和 i_nlink。

i_count 的作用是当一个文件被一个进程引用时，它的数值会加 1，也就是说它记录的是文件被进程引用的次数。i_nlink 的作用则是记录文件产生硬链接的个数。Linux 系统只有在两个数值都清零的时候，文件才被系统认为是删除的。如果我们执行了 rm -rf，却并没有把 i_count 删除，假设此时删除文件有进程在使用，那么它（i_count）数值不为 0。这个时候就是文件看似被删除，但在操作系统还是能便捷的恢复回来。

这就是第 1 种场景删除数据能够被找回的原因（由于 i_count 不为 0）。

第 2 种场景是将没有被进程使用的文件误删除，此时 i_count 和 i_nlink 都为 0。这个时候文件的 inode 连接信息已经被删除了，我们就需要通存放文件的 block 单元，做数据块的数据找回。在系统上我们能看到的文件内容包括：文件名、文件大小、内容，但实际上它的存储依赖两个非常重要的单元，一个是 inode，它用于存放文件的相关元数据，它的元数据里会有一个类似于索引的值，能够索引到后面具体存放数据的 block 单元， block 是一个数据块，用来实际存放数据。我们在删除文件时，其实是把 inode 的链接删除了，但是 block 数据块，并没有删除。

所以这个时候我们依然可以通过分析后端的 block 块，对文件进行恢复。因为 block 块保存着真实的数据，理论上可以作完整的找回数据，不过有一个风险：如果有进程在不断往磁盘写数据时，需要申请新的 block 块，如果操作系统分配已删除文件的 block 块时，那么新的写入数据就会覆盖 block 原来的数据，这时就会造成数据真正丢失的风险。


所以，如果出现这样场景造成数据误删除，需要第一时间 umount 目录所在的磁盘设备。如果没有其他进程在不断地往同一个磁盘块（block）里写数据，那么你的数据理论上还是在 block 块里面，依然可以通过相关分析把数据找回。


这就是我们为什么可以在这两个场景中把数据找回的原因，那么接下来我将讲解如何来恢复数据。我会通过两个案例来进行演示。

## 案例演示

我们先演示第 1 种场景，第 1 种场景是文件在被进程使用过程中被删除，这种场景该如何去恢复文件呢？

首先我登录到测试环境的机器上，这里开启了两个窗口，第 1 个窗口我登录到了这台服务器上，cd /test 目录下，echo 一个测试文件（我把它命名为 DeleteFile），然后把这个内容（"Delete file"）重定向到本地的 deletefile.txt。这个时候我的测试文件就已经生成了。接下来我要做的是开启一个进程，让它实时地使用这个文件。

![](/static/image/Cgq2xl6VdFKAISawAAFE37oeL_w498.png)


这里我使用 tail 命令，持续地查看并且保持监听并使用这个文件。


接下来在另一个窗口，我同样到/test 目录下，而此时我要执行的是 rm -rf ./deletefile.txt，这样就“彻底”把这个文件删除。接下来我们通过 ls，可以看到本地已经没有这个文件了。

![](/static/image/Cgq2xl6VdIqAWXFLAADmb9zygRM543.png)

现在我们已经模拟出文件在进程使用过程中被删除的场景，那么接下来我们来演示恢复该文件。

首先需要找到是哪个进程在使用这个文件，我们可以通过 lsof 命令，grep 刚刚删除的文件名称（deltefile.txt），会列出当前使用文件的进程。我们会看到tail 命令正在使用，它（进程）的 pid 是 4701。

![](/static/image/Ciqah16VdFKAC0qqAADFpBm8jCA754.png)

接下来我们要根据这条线索去恢复数据。我们知道该进程会有使用的文件句柄，那么我们对该进程的文件句柄目录进行查找，cd 到 /proc/{pid}/fd 目录下（这里 pid 为4701），我们到这个目录下，输入 ls -l 命令，这个时候我们会看到，使用这个文件（/test/deletefile.txt）并且它的文件句柄为 3。

![](/static/image/Cgq2xl6VdFKAHdwFAAEiLPkGCz8212.png)

接下来我们要想办法把这个文件进行恢复，输入cp 3 /opt/recovertest/deletefile.txt_bak，这时我就把这 3 个文件做了一个拷贝，实现将数据恢复到 /opt/recovertest/deletefile.txt_bak 文件。



这个时候cd /opt/recovertest/，cat deletefile.txt_bak 看一下里面的内容，可以发现这个文件的内容与刚刚生成的的测试文件内容一致，所以刚刚删除的文件恢复完毕。

![](/static/image/Ciqah16VdL6ATGvHAAEkok5hmU0511.png)

接下来我来演示误删数据场景2（在没有进程使用文件的情况下，如何恢复误删的文件）。演示这种场景，保险起见我在本地多挂载了一块 SDB 的独立硬盘设备。

这种情况要如何恢复数据呢？我们需要安装 extundelete 这个工具。登录到我的测试机上，在这个演示场景里，挂载一块独立硬盘设备 /dev/sdb 并作数据格式化。完成格式化后。把单独的 sdb 设备，挂载到 test 目录下（mount /dev/sdb /test），接下来在 test 目录下生成一个内容为“deletetest ”的测试文件file(echo 'deletetest'>file),这个时候本地目录会生成一个测试的文件：file，再新建一个叫 testdir 的目录（mkdir /test/testdir），那么这时本地既有文件又有目录，也就是我接下来要演示删除的这些文件。

![](/static/image/Ciqah16VdFOAZFR6AAGnNj1Lipk925.png)

我们可以通过 rm -rf ./*，直接把当前目录下的文件整体删除。然后我需要恢复这个文件，原理就是：通过分析它的 block 块，来恢复 inode 链接，要分析并恢复已删除文件的链接，我们要用到一些工具，这里推荐你使用一个叫 extundelete 的命令，它是在 Linux 下基于 ext3\ext4 的文件分析工具，可以对文件系统已删除的文件进行分析，并进行数据恢复。

![](/static/image/Cgq2xl6VdFOAPrHcAAEVDgSoHww503.png)

在执行命令extundelete之前需要先做的是 umount，把我们刚刚误删的目录 umount 掉（umount  /test -l），避免有新的进程再往磁盘块里写数据，同时也便于执行工具进行接下来的分析。

附：extundelete 命令安装方式：



```
yum -y install bzip2 e2fsprogs e2fsprogs-devel gcc-c++

wget https://nchc.dl.sourceforge.net/project/extundelete/extundelete/0.2.4/extundelete-0.2.4.tar.bz2

tar jxvf extundelete-0.2.4.tar.bz2 

cd extundelete-0.2.4

./configure 

make && make install
```

安装好这个工具（extundelete）后，执行：extundelete /dev/sdb --inode 2

我们可以在命令后面加入设备名称，然后加入上面的 inode 进行分析。完成之后我们会看到显示屏幕上已经出现了刚刚删除的文件、名称及目录，还会看到 inode 号以及当前的状态。

![](/static/image/Ciqah16VdOeAOn_dAAB14_xLtZU663.png)

我们也可以选择恢复单独文件类型文件，执行：extundelete /dev/sdb --restore-file file

加入的选项是 --restore-file，后面加你想恢复的文件名称。

在执行以上恢复操作之前，我先要确保数据恢复的目录 /opt/recovertest 下，cd  /opt/recovertest 目录下，执行想恢复的文件 extundelete /dev/sdb --restore-file file。

![](/static/image/Ciqah16VdFOACsNKAAF1sVuVggU217.png)


执行完命令后，会有一个成功的提示。此时在当前目录的 RECOVERED_FILES 目录，有对应恢复好的文件，一个是 file，一个是 file.v1（这个为刚恢复的文件），为什么是 file.v1 呢？因为我在做操作的时候有操作过两遍，所以它恢复了两个文件。第 1 个 file 是我之前写入的内容，第 2 个 file 则是由于我执行了第 2 次恢复，恢复的文件虽然也是 file，所以会自动命名成一个新的版本，叫作 file.v1（这个文件就是我们想要恢复的文件名称）。

![](/static/image/Cgq2xl6VdFOASIv2AAEQM7CcbS4528.png)

刚刚讲到的选项是恢复单个文件，假设我们要恢复所有文件的话，就把选项改为 --restore-all，这样就把分析出来的已删除文件进行了恢复。如果件，只想恢复某一个目录，就可以把 "all" 改成 directory，然后用 restore-directory 这种方式恢复单个已删除的文件目录。



以上就是通过 extundelete 作场景 2 恢复演示。



平时工作中，你还是需要谨慎进行操作系统指令，以避免产生文件系统误删的情况，毕竟恢复起来对我们的业务影响，还有数据风险都是存在的。

