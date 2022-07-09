---
title: git为什么不擅长处理大文件
description: 
published: true
date: 2022-07-09T08:12:13.239Z
tags: git, lfs
editor: markdown
dateCreated: 2022-06-23T09:46:11.915Z
---

## 大型git仓库产生原因

-   它们积累了非常非常长的历史（项目在一个非常长的时间段内成长，包袱不断累积它们包括巨大的二进制资产，需要被跟踪并与代码配对在一起。...也可能是两者都有。）  
    
-   有时，第二种类型的问题会因为旧的、被淘汰的二进制工件仍然存储在资源库中而变得更加复杂。但是有一个相当简单的--虽然很烦人--的解决方法  
    

## 解决方法

-   Git浅层克隆

要实现快速克隆，节省开发者和系统的时间和磁盘空间，第一个解决方案就是只复制最近的修订。Git的浅层克隆选项允许你只提取回购历史中最新的n个提交。

只需要使用--depth选项

```
git clone --depth [depth] [remote-url]
```

**替代浅层克隆的方法：**只克隆一个分支从git 1.7.10开始，你也可以通过克隆单个分支来限制你克隆的历史数量，比如说。

```
git clone [远程地址] --branch [branch_name] --single-branch [folder]
```

-   Git过滤分支

对于那些有很多错误提交的二进制残渣，或者不再需要的旧资产的庞大仓库，一个很好的解决方案是使用git filter-branch。该命令可以让你浏览整个项目的历史，根据预定义模式过滤掉、修改和跳过文件。

一旦你确定了你的 repo 在哪里是重灾区，它就是一个非常强大的工具。有一些辅助脚本可以用来识别大的对象，所以这部分应该是很容易的。语法是这样的

```
git filter-branch --tree-filter 'rm -rf [/path/to/spurious/asset/folder]'
```

git filter-branch有一个小缺点：一旦你使用\_filter-branch\_，你就有效地改写了整个项目的历史。也就是说，所有的提交ID都会改变。这就要求每个开发者重新克隆更新的版本库。

因此，如果你打算用git filter-branch来进行清理操作，你应该提醒你的团队，在操作进行时计划一个短暂的冻结，然后通知大家应该重新克隆版本库。

## 管理有巨大二进制资产的存储库

第二种类型的大资源库是那些有巨大二进制资产的资源库。这是许多不同类型的软件（和非软件！）团队遇到的问题。游戏团队需要处理巨大的3D模型，网页开发团队可能需要跟踪原始图像资产，CAD团队可能需要处理和跟踪二进制交付物的状态。

Git在处理二进制资产方面不是特别差，但也不是特别好。默认情况下，Git 会压缩并存储所有后续的二进制资产的完整版本，如果你有很多二进制资产，这显然不是最佳选择。

有一些基本的调整可以改善这种情况，比如运行垃圾收集（'git gc'），或者在.gitattributes中对某些二进制类型的delta commits的用法进行调整。

但重要的是要反思你项目的二进制资产的性质，因为这将帮助你确定获胜的方法。例如，这里有一些要点需要考虑。

**对于变化很大的二进制文件--而不仅仅是一些元数据头--delta压缩可能是无用的。所以对这些文件使用 "delta off"，以避免不必要的delta压缩工作作为重新打包的一部分。**

在上面的情况下，很可能这些文件的zlib压缩效果也不是很好，所以你可以用 "core.compression 0 "或 "core.loosecompression 0 "关闭压缩。这是一个全局设置，会对所有非二进制文件产生负面影响，而这些文件实际上压缩得很好，所以如果你把二进制资产分割到一个单独的资源库中，这就有意义了。

重要的是要记住，'git gc'将 "重复的 "松散对象变成一个单一的包文件。但同样地，除非文件以某种方式压缩，否则这可能不会对产生的打包文件产生任何重大影响。

探索 "core.bigFileThreshold "的调整。任何大于512MB的文件都不会被delta压缩（无需设置.gitattributes），所以这也许是值得调整的地方。

大文件夹树的解决方案：git sparse-checkout

Git的稀疏签出选项（自Git 1.7.0起可用）对二进制资产问题有轻微帮助。这种技术可以通过明确说明你要填充哪些文件夹来保持工作目录的干净。不幸的是，它并不影响整个本地仓库的大小，但如果你有一棵巨大的文件夹树，那就很有帮助。

涉及的命令是什么？下面是一个例子。

克隆一次完整的版本库：'git clone'。

激活该功能：'git config core.sparsecheckout true

明确添加需要的文件夹，忽略assets文件夹。

echo src/ ' .git/info/sparse-checkout

按照规定读取树。

完成上述工作后，你可以回去使用正常的 git 命令，但你的工作目录将只包含你上面指定的文件夹。

## git lfs原理

Git 是一个分布式的版本控制系统，这意味着在克隆过程中，整个仓库的历史都会传输给客户端。对于包含大文件的项目，尤其是经常修改的大文件，这种初始克隆会花费大量的时间，因为每个文件的每个版本都要由客户端下载。Git LFS（大文件存储）是由Atlassian、GitHub和其他一些开源贡献者开发的Git扩展，它通过懒散地下载大文件的相关版本来减少仓库中大文件的影响。具体来说，大文件在签出过程中被下载，而不是在克隆或获取过程中。

Git LFS通过用微小的指针文件替换仓库中的大文件来做到这一点。在正常使用过程中，你永远不会看到这些指针文件，因为它们是由 Git LFS 自动处理的。

1.  当你添加一个文件到你的仓库时，Git LFS 会将其内容替换成一个指针，并将文件内容存储在本地的 Git LFS 缓存中。

![Untitled Diagram.drawio.png](https://ucc.alicdn.com/pic/developer-ecology/9edefb71f1884b798a30aa0b7c0ccd90.png "Untitled Diagram.drawio.png")

2.  当你推送新的提交到服务器时，新推送的提交所引用的任何 Git LFS 文件会从本地的 Git LFS 缓存转移到与你的 Git 仓库绑定的远程 Git LFS 存储。

![GITLFS2.drawio.png](https://ucc.alicdn.com/pic/developer-ecology/954c2446b5cf49b18f110b278dd931e7.png "GITLFS2.drawio.png")

3.  当你签出一个包含Git LFS指针的提交时，它们会被替换成本地Git LFS缓存中的文件，或者从远程Git LFS存储中下载。

![GITLFS3.drawio.png](https://ucc.alicdn.com/pic/developer-ecology/00884442d42348c1994e4130d529fa0a.png "GITLFS3.drawio.png")

Git LFS是无缝的：在你的工作副本中，你将只看到你的实际文件内容。这意味着你可以在不改变现有的Git工作流程的情况下使用Git LFS；你只需、编辑、、和正常工作。而且操作会明显加快，因为你只下载你实际签出的提交所引用的大文件的版本，而不是曾经存在的文件的每个版本。

## 参考链接

-   [Large and In Charge - using Git LFS (testdouble.com)](https://blog.testdouble.com/posts/2017-01-25-how-and-why-to-use-git-lfs/)  
    
-   [How to handle big repositories with Git | Atlassian Git Tutorial](https://www.atlassian.com/git/tutorials/big-repositories)