---
title:      " Linux find命令教程：15个find命令用法 "
excerpt: "在系统上查找文件或目录时，Linux上的find命令无与伦比。它使用简单，而且有许多不同的选项，可让您微调文件搜索。继续阅读以查看如何使用此命令在系统上查找任何内容的示例。一旦您知道如何在Linux中使用find命令，每个文件都只需敲击几下。"
date:       2019-12-20
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Linux ]

---

# 查找目录

您可以使用-type d选项告诉find命令专门查找目录。这将使find命令仅搜索匹配的目录名，而不搜索文件名。
```bash
$ find /path/to/search -type d -name "name-of-dir"
```

#查找隐藏文件

由于Linux中的隐藏文件和目录以句点开头，因此我们可以在搜索字符串中指定此搜索模式，以便递归列出隐藏的文件和目录。
```bash
$ find /path/to/search -name ".*"
```

#查找特定大小或大于X的文件

find的-size选项允许我们搜索特定大小的文件。它可用于查找确切大小的文件，大于或小于特定大小的文件或适合指定大小范围的文件。以下有些例子：
搜索大于10MB的文件：

```bash
$ find /path/to/search -size +10M
```
搜索小于10MB的文件：
```bash
$ find /path/to/search -size -10M
```
搜索大小恰好为10MB的文件：
```bash
$ find /path/to/search -size 10M
```
搜索大小在100MB到1GB之间的文件：
```bash
$ find /path/to/search -size +100M -size -1G
```

# 从文件列表中查找

如果您有需要搜索的文件列表（例如，在.txt文件中），则可以使用find和grep命令的组合来搜索文件列表。为了使此命令起作用，只需确保要搜索的每个模式之间都用换行符隔开。
```bash
$ find /path/to/search | grep -f filelist.txt
```
grep的-f选项表示“file”，并允许我们指定要匹配的字符串文件。这导致find命令返回与列表中的文件或目录名称匹配的任何文件或目录名称。


# 不在列表中查找

使用上一个示例中提到的相同文件列表，您还可以使用find来搜索与文本文件内的模式不符的任何文件。再一次，我们将结合使用find和grep命令；我们只需要用grep指定一个附加选项：
```bash
$ find /path/to/search | grep -f filelist.txt
```
grep的-v选项表示“逆向匹配”，并且将返回与文件列表中指定的任何模式都不匹配的文件列表。


# 设置maxdepth

find命令默认将进行递归搜索。这意味着它将在指定的目录中搜索您指定的模式，以及您告诉它要搜索的目录中的所有子目录。
例如，如果告诉find搜索Linux（/）的根目录，则无论存在多少个子目录，它都会搜索整个硬盘。您可以使用-maxdepth选项来规避此行为。
在-maxdepth之后指定一个数字，以指示查找应递归搜索的子目录数。
仅搜索当前目录中的文件，而不递归搜索：
```bash
$ find . -maxdepth 0 -name "myfile.txt"
```
仅在当前目录和更深的一个子目录中搜索文件：
```bash
$ find . -maxdepth 1 -name "myfile.txt"
```

# 查找空文件（零长度）

要使用find搜索空文件，可以使用-empty标志。搜索所有空文件：
```bash
$ find /path/to/search -type f -empty
```
搜索所有空目录：
```bash
$ find /path/to/search -type d -empty
```
如果希望自动删除find返回的空文件或目录，那么将此命令与-delete选项结合使用也非常方便。
删除目录（和子目录）中的所有空文件：
```bash
$ find /path/to/search -type f -empty -delete
```

# 查找最大的目录或文件

如果您想快速确定系统上哪些文件或目录占用了最多的空间，则可以使用find进行递归搜索，并按文件和目录的大小输出排序的列表。
如何显示目录中最大的文件：
```bash
$ find /path/to/search -type f -printf "%s\t%p\n" | sort -n | tail -1
```
请注意，find命令已被排序到另外两个方便的Linux实用程序：sort和tail。 Sort将按文件的大小顺序排列文件列表，而tail将仅输出列表中的最后一个文件，该文件也是最大的。
如果您要输出例如最大的前5个文件，则可以调整tail命令。
```bash
$ find /path/to/search -type f -printf "%s\t%p\n" | sort -n | tail -5
```
或者，您可以使用head命令来确定最小的文件：
```bash
$ find /path/to/search -type f -printf "%s\t%p\n" | sort -n | head -5
```
如果要搜索目录而不是文件，只需在类型选项中指定“ d”即可。如何显示最大目录：
```bash
$ find /path/to/search -type d -printf "%s\t%p\n" | sort -n | tail -1
```

# 查找setuid设置文件

Setuid是“set user ID on execution”的缩写，它是一种文件权限，允许普通用户运行具有升级特权（例如root）的程序。
出于明显的原因，这可能是一个安全问题，但是可以使用find命令和一些选项轻松隔离这些文件。
find命令有两个选项可帮助我们搜索具有特定权限的文件：-user和-perm。要查找普通用户能够以root特权执行的文件，可以使用以下命令：
```bash
$ find /path/to/search -user root -perm /4000
```
![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-20/setuid.png)

在上面的屏幕截图中，我们包含了-exec选项，以便显示有关查找返回文件的更多输出。整个命令如下所示：
```bash
$ find /path/to/search -user root -perm /4000 -exec ls -l {} \;
```
您也可以在此命令中用“ root”代替您要作为所有者搜索的任何其他用户。或者，您可以搜索具有SUID权限的所有文件，而根本不指定一个用户：
```bash
$ find /path/to/search -perm /4000
```

# 查找sgid设置文件

查找具有SGID设置的文件与查找具有SUID的文件几乎相同，只是需要将4000的权限更改为2000：
```bash
$ find /path/to/search -perm /2000
```
您还可以通过在perms选项中指定6000来搜索，同时设置了SUID和SGID的文件：
```bash
$ find /path/to/search -perm /6000
```

# 列出文件未经允许被拒绝

使用find命令搜索文件时，您必须对要搜索的目录和子目录具有读取权限。如果您没有找到，find将输出一条错误消息，但会继续浏览您确实拥有权限的目录。
没有权限尽管这可能发生在许多不同的目录中，但在搜索根目录时肯定会发生。
这意味着，当您尝试在整个硬盘上搜索文件时，find命令将产生大量错误消息。
为避免看到这些错误，您可以将find的stderr输出重定向到stdout，并将其通过管道传递到grep。
```bash
$ find / -name "myfile.txt" 2>%1 | grep -v "Permission denied"
```
此命令使用grep的-v（反向）选项来显示所有输出，除了显示“拒绝权限”之外的所有输出。

# 在最近X天内查找修改过的文件

使用find命令上的-mtime选项搜索最近X天内被修改的文件或目录。它也可以用于搜索X天之前的文件，或X天之前被完全修改过的的文件。
以下是一些如何在find命令上使用-mtime选项的示例：
搜索最近30天内修改过的所有文件：
```bash
$ find /path/to/search -type f -mtime -30
```
搜索超过30天之前已修改的所有文件：
```bash
$ find /path/to/search -type f -mtime +30
```
搜索30天前刚修改过的所有文件：
```bash
$ find /path/to/search -type f -mtime 30
```
如果希望find命令输出有关找到的文件的更多信息，例如修改日期，则可以使用-exec选项并包含ls命令：
```bash
$ find /path/to/search -type f -mtime -30 -exec ls -l {} \;
```

# 按时间排序

要按文件的修改时间对查找结果进行排序，您可以使用-printf选项以可排序的方式列出时间，然后将其输出到sort实用程序。
```bash
$ find /path/to/search -printf "%T+\t%p\n" | sort
```
此命令将对旧的文件进行排序。如果您希望较新的文件首先显示，只需传递-r（反向）选项即可进行排序。
```bash
$ find /path/to/search -printf "%T+\t%p\n" | sort -r
```

# 定位和查找之间的区别

Linux上的locate命令是搜索系统上文件的另一种好方法。它没有像find命令那样包含过多的搜索选项，因此它的灵活性较差，但仍然很方便。
```bash
$ locate myfile.txt
```
locate命令通过搜索包含系统上所有文件名的数据库来工作。搜索到的数据库已使用upatedb命令进行更新。
由于locate命令不必实时搜索系统上的所有文件，因此它比find命令效率更高。但是，除了缺少选项之外，还有另一个缺点：文件数据库每天仅更新一次。
您可以通过运行updatedb命令手动更新此文件数据库：
```bash
$ updatedb
```
当您需要在整个硬盘驱动器中搜索文件时，locate命令特别有用，因为find命令自然需要更长的时间，因为它必须实时遍历每个目录。
如果搜索一个特定目录（已知其中不包含大量子目录），则最好坚持使用find命令。

# find命令的CPU负载

在搜索大量目录时，find命令可能会占用大量资源。它本来应该允许更重要的系统进程具有优先级，但是如果需要确保find命令占用生产服务器上的较少资源，则可以使用ionice或nice命令。
监视find命令的CPU使用情况：
```bash
$ top
```
降低find命令的输入/输出优先级：
```bash
$ ionice -c3 -n7 find /path/to/search -name "myfile.txt"
```
降低find命令的CPU优先级：
```bash
$ nice -n 19 find /path/to/search -name "myfile.txt"
```
或结合使用这两个实用程序以真正确保低I / O和低CPU优先级：
```bash
$ nice -n ionice -c2 -n7 find /path/to/search -name "myfile.txt"
```