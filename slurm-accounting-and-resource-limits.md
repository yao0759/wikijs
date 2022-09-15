---
title: slurm--核算和资源限制
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出
published: true
date: 2022-09-15T13:52:34.510Z
tags: slurm, hpc
editor: markdown
dateCreated: 2022-09-15T10:11:48.437Z
---

# Slurm--核算和资源限制

## 概览

Slurm可以被配置为收集每个作业和作业步骤执行的核算信息。核算记录可以被写入一个简单的文本文件或一个数据库。目前正在执行的作业和已经终止的作业的信息都是可用的。sacct命令可以报告正在运行或已经终止的作业的资源使用情况，包括单个任务，这对于检测任务之间的负载不平衡非常有用。sstat命令可用于仅对当前正在运行的作业进行统计。它也可以为你提供关于任务之间不平衡的有价值的信息。sreport可以用来生成基于特定时间间隔内执行的所有作业的报告。

有三种不同的插件类型与资源核算有关。与这些插件相关的Slurm配置参数（在slurm.conf中）包括：

- AccountingStorageType控制如何记录详细的作业和作业步骤信息。你可以把这些信息存储在一个文本文件或SlurmDBD中。
- JobAcctGatherType与操作系统有关，它控制了使用什么机制来收集核算信息。支持的值是jobacct_gather/linux, jobacct_gather/cgroup和jobacct_gather/none（不收集信息）。
- JobCompType控制工作完成信息的记录方式。这可以用来记录基本作业信息，如作业名称、用户名、分配的节点、开始时间、完成时间、退出状态等。如果只需要保存基本作业信息，这个插件应该能满足你的需求，而且开销最小。你可以将这些信息存储在一个文本文件，或者MySQL或MariaDB数据库中。

使用sacct查看工作信息，取决于AccountingStorageType被配置为收集和存储该信息。sreport的使用取决于一些数据库被用来存储这些信息。

使用 sacct 或 sstat 来查看作业中的资源使用信息，取决于 JobAcctGatherType 和 AccountingStorageType 被配置为收集和存储该信息。

将核算信息存储到文本文件中是非常简单的。只要配置适当的插件（如JobCompType=jobcomp/filetxt），然后指定文件的路径名（如JobCompLoc=/var/log/slurm/job_completions）。使用logrotate或类似工具，防止日志文件过大。在移动文件后，但在压缩文件前，向slurmctld守护进程发送一个SIGUSR2信号，这样就会有新的日志文件产生。

将数据直接从Slurm中存储到数据库中似乎很有吸引力，但它不仅需要为Slurm控制守护进程（slurmctld）提供用户名和密码数据，还需要为需要访问数据的用户命令（sacct、sreport和sacctmgr）提供。将潜在的敏感信息提供给所有用户，使得数据库的安全性更难提供。通过一个中间守护程序发送数据可以提供更好的安全性和性能（通过缓存数据）。SlurmDBD（Slurm Database Daemon）提供了这样的服务。SlurmDBD是用C语言编写的，多线程，安全且快速。下面将介绍使用SlurmDBD所需的配置。直接将信息存储到数据库中的做法类似于

注意，SlurmDBD依赖于现有的Slurm插件来进行身份验证，以及Slurm SQL来使用数据库，但在安装SlurmDBD的主机上不需要其他的Slurm命令和守护程序。在要运行SlurmDBD的服务器上安装slurm和slurm-slurmdbd RPMs。

注意，如果你从使用MySQL插件切换到使用SlurmDBD插件，你必须确保集群已经被添加到数据库中。MySQL插件没有这个要求，但如果你在使用MySQL插件时有这个要求，也不会有什么影响。你可以通过以下方式验证

```
sacctmgr list cluster
```

如果集群不在那里，就添加它（其中我的集群的名字是snowflake）。

```
sacctmgr add cluster snowflake
```

如果不这样做，将导致slurmctld在切换后无法与slurmdbd对话。如果你打算升级到新版本的Slurm，不要同时切换插件，否则你可能得到意想不到的结果。先做一个再做另一个。

如果SlurmDBD被配置为使用但没有响应，那么slurmctld将利用一个内部缓存，直到SlurmDBD返回服务。缓存的数据在关机时由slurmctld写入本地存储，并在启动时恢复。如果SlurmDBD在slurmctld启动时不可用，将使用基于守护进程最后一次通信时状态的有效银行账户、用户限额等的缓存。注意，SlurmDBD必须在slurmctld首次启动时进行响应，因为没有这种关键数据的缓存。由slurmctld生成的作业和步骤记录将根据需要写入缓存，并在返回服务时传输给SlurmDBD。注意，如果SlurmDBD宕机的时间足够长，排队记录的数量超过了最大队列大小，那么消息将开始被丢弃。

## 架构

通过SlurmDBD，我们能够在一个地方收集多个集群的数据。这确实对用户的命名和ID施加了一些限制。核算是通过用户名（而不是用户ID）来维护的，但是一个给定的用户名应该在所有的计算机上指的是同一个人。认证依赖于用户ID号码，所以这些号码必须在与每个SlurmDBD通信的所有计算机上统一，至少对需要认证的用户来说是如此。特别是，配置的SlurmUser必须在所有集群中具有相同的名称和ID。如果你计划有用户账户、限制等的管理员，他们也必须在所有集群中拥有一致的名称和ID。如果你计划限制对核算记录的访问（例如，只允许一个用户查看他的工作记录），那么所有用户都应该有一致的名字和ID。

注意：只支持小写的用户名。

确保数据安全的最好方法是对SlurmDBD的通信进行认证，我们推荐MUNGE来实现这一目的。如果你有一个由Slurm管理的集群，并在这一个集群上执行SlurmDBD，正常的MUNGE配置就足够了。否则，MUNGE应该被安装在所有Slurm管理的集群的所有节点上，加上执行SlurmDBD的机器。然后，你可以选择为所有这些计算机安装一个MUNGE密钥，或者为每个集群维护一个唯一的密钥，再加上集群之间通信的第二个密钥，以提高安全性。MUNGE的改进计划是在一个配置文件中支持两个密钥，但目前必须用不同的配置启动两个不同的守护程序，以支持两个不同的密钥（创建两个密钥文件，用--密钥文件选项启动守护程序，以找到适当的密钥，再加上--套接字选项，为每个密钥指定不同的本地域套接字）。在Slurm和SlurmDBD配置文件（分别为slurm.conf和slurmdbd.conf，更多细节将在下面提供）中需要本地域套接字的路径名。

无论你是否使用任何认证模块，你都需要有一种方法让SlurmDBD为用户和/或管理员获得UID。如果使用MUNGE，最理想的是你的用户在所有的集群上都有相同的ID。如果是这样的话，你应该在数据库服务器上有一个每个集群的/etc/passwd文件的组合，以允许DBD解析名字进行认证。如果使用MUNGE，而用户的名字不在passwd文件中，行动将失败。如果不使用MUNGE，你应该把任何你想成为管理员或操作员的人加入到passwd文件中。如果他们打算运行sacctmgr或任何核算工具，他们应该有相同的UID，否则他们将无法正确认证。一个LDAP服务器也可以作为收集这些信息的途径。

## Slurm JobComp配置

目前，SlurmDBD不支持作业完成，但可以直接写入数据库、脚本或平面文件。如果你正在使用核算存储插件运行，使用作业完成插件可能是多余的。如果你想对此进行配置，一些比较重要的参数包括。

- JobCompHost：只有在使用数据库时才需要。数据库服务器执行的主机的名称或地址。
- JobCompLoc：只有在使用平面文件时才需要。写入作业完成数据的文件的位置。
- JobCompPass：只有在使用数据库时才需要。连接到数据库的用户的密码。由于密码不能被安全地维护，不建议直接将信息存储在数据库中。
- JobCompPort：只有在使用数据库时才需要。数据库接受通信的网络端口。
- JobCompType：设置为 "jobcomp/mysql "或 "jobcomp/filetxt "的jobcomp插件的类型。
- JobCompUser：只有在使用数据库时才需要。用来连接数据库的用户名。
- JobCompParams：传递任意的文本字符串给作业完成插件。

## 构建前的Slurm核算配置

虽然SlurmDBD可以用一个平面文本文件来记录工作完成情况和类似的数据，但这种配置不允许在用户和账户之间建立 "关联"。一个数据库允许这样的配置。

MySQL或MariaDB是首选的数据库。

注意：如果你有一个现有的Slurm核算数据库，并计划将你的数据库服务器从10.2.1之前的版本升级到MariaDB 10.2.1（或更新的版本），或从任何版本的MySQL，请联系SchedMD寻求帮助。

要启用这种数据库支持，人们只需要在系统上拥有他们希望使用的数据库的开发包。Slurm使用MySQL中的InnoDB存储引擎，使回滚成为可能。这必须在你的MySQL安装中可用，否则回滚将无法工作。

slurm配置脚本使用mysql_config来查找它需要的关于已安装的库和头文件的信息。在配置slurm构建时，你可以用--with-mysql_conf=/path/to/mysql_config选项指定你的mysql_config脚本的位置。在一个成功的配置中，输出是这样的。

```
checking for mysql_config... /usr/bin/mysql_config
MySQL test program built properly.
```

注意：在第一次运行slurmdbd之前，查看MySQL的innodb_buffer_pool_size的当前设置。考虑将这个值设置得足够大，以处理数据库的大小。当把大表转换到新的数据库模式或清除旧记录时，这个值太小会有问题。我们建议将系统内存的很大一部分分配给它，记住运行MySQL/MariaDB的机器上的其他资源需求，大约在可用内存的5%到50%之间。也建议将innodb_lock_wait_timeout和innodb_log_file_size设置为比默认值大的值。

请看下面的例子。

```
mysql> SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
+-------------------------+------------+
| Variable_name           | Value      |
+-------------------------+------------+
| innodb_buffer_pool_size | 4294967296 |
+-------------------------+------------+
1 row in set (0.001 sec)


$cat my.cnf
...
[mysqld]
innodb_buffer_pool_size=4096M
innodb_log_file_size=64M
innodb_lock_wait_timeout=900
...
```

此外，在5.7之前的MySQL版本中，默认的行格式被设置为COMPACT，这可能会在升级期间创建表时造成一些问题。在最近的版本中，它被改变为动态格式。表的行格式决定了其行在页面中的物理存储方式，并直接影响到查询和DML操作的性能。在非常特殊的情况下，使用DYNAMIC以外的格式可能会导致行不适合放入页面，MySQL可能会因此在创建表的过程中抛出一个错误。因此，建议在创建你的数据库表之前，如果你默认不使用DYNAMIC，请仔细阅读关于行的格式，如果你的数据库版本支持，请考虑设置。如果在升级过程中出现以下InnoDB错误，这时可以对表进行修改（可能需要一些时间），将行格式设置为DYNAMIC，以便让转换继续进行。

```
[Warning] InnoDB: Cannot add field ... in table ... because after adding it, the row size is Y which is greater than max
```

你可以通过显示innodb_default_row_format变量看到默认的行格式是什么。

```
mysql> SHOW VARIABLES LIKE 'innodb_default_row_format';
+---------------------------+---------+
| Variable_name             | Value   |
+---------------------------+---------+
| innodb_default_row_format | dynamic |
+---------------------------+---------+
1 row in set (0.001 sec)
```

你也可以通过运行以下命令看到表是如何被创建的，其中db_name是你在slurmdbd.conf中设置的Slurm数据库（StorageLoc）的名称。

```
mysql> SHOW TABLE STATUS IN db_name;
```

## 构建后的Slurm核算配置

为了简单起见，我们将在假设你是用SlurmDBD运行的情况下进行。你可以直接与一个存储插件进行通信，但这提供了最小的安全性。

必须设置几个Slurm配置参数以支持在SlurmDBD中归档信息。SlurmDBD有一个单独的配置文件，它在一个单独的章节中被记录。请注意，你可以将核算信息写入SlurmDBD，而作业完成记录则写入文本文件或根本就不维护。如果你不设置以 "AccountingStorage "开头的配置参数，那么核算信息将不会被引用或记录。

- AccountingStorageEnforce: 这个选项包含一个逗号分隔的选项列表，你可能想强制执行。有效的选项是以下任何逗号分隔的组合

  - associations 如果用户的关联不在数据库中，这将阻止用户运行作业。这个选项将防止用户访问无效的账户。
  - limits - 这将强制执行设置在关联和qos上的限制。通过设置这个选项，"关联 "选项被自动设置。如果使用qos，限制将被强制执行，但如果你想强制访问qos，仍然需要下面描述的'qos'。
  - nojobs - 这将使得没有工作信息被存储在核算中。通过设置它，'nosteps'也被设置。
  - nosteps - 这将使核算中不存储步骤信息。nojobs和nosteps在你想使用限制但并不真正关心利用率的环境中都很有用。
  - qos - 这将要求所有作业指定（公开地或默认地）一个有效的qos（服务质量）。QOS值是为数据库中的每个关联定义的。通过设置这个选项，"关联 "选项被自动设置。如果你想强制执行QOS限制，你需要使用'限制'选项。
  - safe - 这将确保作业只有在使用设置了GrpTRESMins限制的关联或QOS时才会被启动，如果该作业能够运行到完成。如果不设置这个选项，只要作业的使用量没有达到TRES-分钟的限制，作业就会被启动，这可能会导致作业被启动，但在达到限制时又被杀死。通过设置这个选项，"关联 "选项和 "限制 "选项都会自动设置。
  - wckeys - 这将防止用户在他们没有权限的wckey下运行作业。通过使用这个选项，"关联 "选项被自动设置。'TrackWCKey'选项也被设置为真。

  **(注意：associations 是集群、账户、用户名和可选分区名称的组合)。**

  **如果没有设置AccountingStorageEnforce（默认行为），作业将根据每个集群上在Slurm中配置的策略来执行。**

- AccountingStorageExternalHost。一个逗号分隔的外部slurmdbds列表（<host/ip>[:port][,...]），用于注册。如果没有给出端口，将使用AccountingStoragePort。这允许在外部slurmdbd注册的集群使用--cluster/-M客户端命令选项互相通信。如果外部slurmdbd不存在，该集群将把自己添加到外部slurmdbd中。如果外部slurmdbd上已经存在一个非外部集群，slurmctld将忽略向外部slurmdbd的注册。

- AccountingStorageHost。SlurmDBD执行的主机的名称或地址

- AccountingStoragePass。如果将SlurmDBD与第二个MUNGE守护进程一起使用，请存储MUNGE用来提供全企业范围内的认证的命名套接字的路径名（即/var/run/munge/moab.socket.2） 。否则将使用默认的MUNGE守护程序。

- AccountingStoragePort: SlurmDBD接受通信的网络端口。

- AccountingStorageType。设置为 "accounting_storage/slurmdbd"。

- ClusterName。设置为每个Slurm管理的集群的唯一名称，以便可以识别每个集群的核算记录。

- TrackWCKey: 布尔值。如果你想跟踪用户的wckeys（工作量特征键）。Wckey是一种正交的方式，针对可能不相关的账户进行核算。当一个作业运行时，使用-wckey选项指定一个值，核算记录将由这个wckey收集。

## SlurmDBD配置

SlurmDBD需要它自己的配置文件，称为 "slurmdbd.conf"。这个文件应该只存在于执行SlurmDBD的计算机上，并且只能由执行SlurmDBD的用户（例如 "slurm"）阅读。这个文件应该被保护起来，防止未经授权的访问，因为它包含了数据库的登录名和密码。参见 "man slurmdbd.conf "以获得更完整的配置参数描述。一些比较重要的参数包括。

- AuthInfo：如果将SlurmDBD与第二个MUNGE守护进程一起使用，请存储MUNGE用来提供企业范围内的命名套接字的路径名。否则将使用默认的MUNGE守护进程。
- AuthType。定义Slurm组件之间通信的认证方法。建议使用 "auth/munge "的值。
- DbdHost。执行Slurm数据库守护程序的机器的名称。这应该是一个没有完整域名的节点名称（例如："lx0001"）。默认为localhost，但应该提供以避免出现警告信息。
- DbdPort：Slurm数据库守护程序（slurmdbd）工作时监听的端口号。默认值是系统建立时的SLURMDBD_PORT。如果没有明确指定，它将被设置为6819。这个值必须等于slurm.conf文件中的AccountingStoragePort参数。
- LogFile: 写入Slurm数据库守护程序日志的文件的完全合格的路径名。默认值是无（通过syslog执行日志）。
- PluginDir: 确定寻找Slurm插件的地方。这是一个用冒号分隔的目录列表，像PATH环境变量。默认值是在配置时给出的前缀+"/lib/slurm"。
- SlurmUser：slurmdbd守护进程执行的用户名称。这个用户必须存在于执行Slurm数据库守护程序的机器上，并且与执行slurmctld的主机具有相同的UID。为了安全起见，建议使用 "root "以外的用户。默认值是 "root"。这个名字也应该是所有向SlurmDBD报告的集群上的同一个SlurmUser。注意：如果这个用户与为slurmctld设置的用户不同，并且不是root，则必须用AdminLevel=Admin将其加入核算，并且必须重新启动slurmctld。

- StorageHost。定义数据库运行的主机名称，我们将在那里存储数据。理想情况下，这应该是SlurmDBD执行的主机，但也可以是另一台机器。

- StorageLoc：指定写入核算记录的数据库的名称。对于数据库来说，默认的数据库是slurm_acct_db。注意名称中不能有'/'，否则将使用默认值。
- StoragePass：定义用于访问数据库的密码，以存储作业核算数据。
- StoragePort：定义数据库的监听端口。
- StorageType：定义核算存储机制。目前唯一可接受的值是 "accounting_storage/mysql"。accounting_storage/mysql "这个值表示核算记录应该写到StorageLoc参数指定的MySQL或MariaDB数据库中。这个值必须被指定。
- StorageUser：定义我们要连接到数据库的用户名称，以存储工作核算数据。

## MySQL配置

注意：如果你有一个现有的Slurm核算数据库，并计划将你的数据库服务器从10.2.1之前的版本升级到MariaDB 10.2.1（或更新的版本），或从任何版本的MySQL，请联系SchedMD寻求帮助。

虽然Slurm会自动创建数据库表，但你需要确保StorageUser在MySQL或MariaDB数据库中被赋予权限，以便这样做。作为mysql用户，使用诸如以下命令授予该用户权限。

GRANT ALL ON StorageLoc.* TO 'StorageUser'@'StorageHost';
(需要打钩)

(你需要以root身份来做这件事。另外，在密码使用信息中，有一行是以'->'开头的。这是一个继续提示，因为之前的mysql语句没有以'；'结束。它假定你希望输入更多信息）。

如果你想让Slurm自己创建数据库，以及任何未来的数据库，你可以将你的授予行改为*.*而不是StorageLoc.*。

例子：

```sql
mysql@snowflake:~$ mysql
Welcome to the MySQL monitor.Commands end with ; or \g.
Your MySQL connection id is 538
Server version: 5.0.51a-3ubuntu5.1 (Ubuntu)

Type 'help;' or '\h' for help. Type '\c' to clear the buffer.

mysql> create user 'slurm'@'localhost' identified by 'password';
Query OK, 0 rows affected (0.00 sec)

mysql> grant all on slurm_acct_db.* TO 'slurm'@'localhost';
Query OK, 0 rows affected (0.00 sec)
```

你可能也需要对系统名称做同样的处理，以便使mysql正确工作。

```sql
mysql> grant all on slurm_acct_db.* TO 'slurm'@'system0';
Query OK, 0 rows affected (0.00 sec)
where 'system0' is the localhost or database storage host.
```

或者使用密码

```sql
mysql> grant all on slurm_acct_db.* TO 'slurm'@'localhost'
    -> identified by 'some_pass' with grant option;
Query OK, 0 rows affected (0.00 sec)
```

案例中也是如此，你做的是对系统名称进行同样的处理。

```sql
mysql> grant all on slurm_acct_db.* TO 'slurm'@'system0'
    -> identified by 'some_pass' with grant option;
where 'system0' is the localhost or database storage host
```

确认支持innodb

```sql
mysql> SHOW ENGINES;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| ...                |         |                                                                |              |      |            |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| ...                |         |                                                                |              |      |            |
+--------------------+---------+----------------------------------------------------------------+-------------
```

创建数据库

```sql
mysql> create database slurm_acct_db;
```

这将授予用户 "slurm "权限，使其在本地主机或存储主机系统上做它需要做的事情。这必须在SlurmDBD正常工作之前完成。在你授予mysql中的用户 "slurm "权限后，你可以启动SlurmDBD和其他Slurm守护程序。你可以通过输入它的路径名"/usr/sbin/slurmdbd "或"/etc/init.d/slurmdbd start "来启动SlurmDBD。你可以通过输入'ps aux | grep slurmdbd'来验证SlurmDBD正在运行。

如果SlurmDBD没有运行，你可以在启动SlurmDBD时使用-v选项来获得更详细的信息。用'-D'选项在守护模式下启动SlurmDBD也可以帮助调试，这样你就不必去看日志来发现问题。

## 工具

有几个工具可以用来处理核算数据，sacct、sacctmgr和sreport。这些工具都是通过SlurmDBD守护程序获取或设置数据。

- sacct用于生成正在运行和已经完成的作业的核算报告。
- sacctmgr用于管理数据库中的关联：添加或删除集群，添加或删除用户，等等。
- sreport用于生成在给定时间段内收集到的各种使用报告。

更多信息请参见每个命令的手册页面。

## 数据库配置

核算记录是根据我们所说的 "Association "来维护的，它由四个元素组成：集群、账户、用户名和一个可选的分区名称。使用 sacctmgr 命令来创建和管理这些记录。

注意：设置核算关联是有顺序的。你必须在添加账户之前定义群集，你必须在添加用户之前添加账户。

例如，要添加一个名为 "snowflake "的集群到数据库中，执行这一行（注意：从20.02版开始，如果集群不存在，slurmctld会在启动时将其添加到数据库中。添加后仍然需要创建关联）。

```
sacctmgr add cluster snowflake
```

将账户 "none "和 "test "添加到集群 "snowflake "中，并执行这样的一行。

```
sacctmgr add account none,test Cluster=snowflake \
  Description="none" Organization="none"
```

如果你有更多的集群想添加这些账户，你可以不指定集群，这将把账户添加到系统中的所有集群，或者在集群选项中用逗号分隔你想添加的集群名称。注意，可以通过逗号分隔名称，同时添加多个账户。必须指定账户的描述和它所属的组织。这些术语以后可以用来生成核算报告。账户可以按层次排列。例如，化学和物理账户可能是科学账户的子女。层次结构可以有一个任意的深度。只要在添加账户行中指定parent=''选项，就可以构建层次结构。对于上面的例子，执行

```
sacctmgr add account science \
 Description="science accounts" Organization=science
sacctmgr add account chemistry,physics parent=science \
 Description="physical sciences" Organization=science
```

使用类似的语法将用户添加到账户中。例如，要允许用户da在所有集群上执行作业，其默认账户为test execute。

```
sacctmgr add user brian Account=physics
sacctmgr add user da DefaultAccount=test
```

如果在集群snowflake的slurm.conf中配置了AccountingStorageEnforce=associations，那么用户da将被允许在账户test和将来添加的任何其他账户中运行。任何试图使用其他账户的行为都会导致作业被中止。如果他没有在作业提交命令中指定一个账户，那么账户test将是默认的。

还可以创建与特定分区绑定的关联。当使用sacctmgr的 "添加用户 "命令时，你可以包括Partition=<PartitionName>选项来创建一个关联，这个关联对于具有相同账户和用户的其他关联是唯一的。

## 集群选项

当添加或修改一个集群时，这些是sacctmgr的可用选项。

- Name=Cluster Name

## 帐户选项

当添加或修改一个帐户时，以下是sacctmgr的选项。

- Cluster= 只将此帐户添加到这些群集。默认情况下，该账户会被添加到所有定义的群组中。

- Description= 账户的描述。(默认为账户名称)

- Name= 账户的名称。注意这个名字必须是唯一的，不能代表账户层次结构中不同的银行账户。
- Organization= 账户的组织机构。(默认为父账户，除非父账户是根账户，那么组织就被设置为账户名称。)
- Parent=使这个账户成为这个其他账户的子账户（已经添加）。

## 用户选项

当添加或修改一个用户时，可以使用以下sacctmgr选项。

- Account= 要添加用户的账户

- AdminLevel= 这个字段用于允许用户为这个用户增加核算权限。有效的选项是

  - None

  - Operator：可以添加、修改和删除任何数据库对象（用户、账户等），并添加其他Operator

    在服务于slurmDBD的slurmctld上，这些用户可以

    - 查看被PrivateData标志阻挡在常规使用之外的信息

    - 创建/改变/删除预订

  - Admin：这些用户在数据库中拥有与操作员相同的权限水平。他们也可以改变服务的slurmctld上的任何东西，就像他们是slurm用户或root一样。

- Cluster= 只添加到这些集群上的账户（默认是所有集群）。

- DefaultAccount= 用户的默认账户，当提交作业时没有指定账户时使用。(创建时需要)

- DefaultWCKey= 用户的默认wckey，在提交作业时没有指定wckey时使用。(只在跟踪wckeys时使用。)

- Name= 用户名称

- NewName= 用来在核算数据库中重新命名一个用户

- Partition= 此关联适用于Slurm分区的名称

## 限制执行

各种限制和限制执行在资源限制网页上有描述。

要启用任何限制执行，你至少要在slurm.conf中设置AccountingStorageEnforce=limits。否则，即使你设置了限制，它们也不会被强制执行。AccountingStorageEnforce的其他选项以及每个选项的解释可在资源限制文件中找到。

## 修改实体

当修改实体时，你可以用类似SQL的方式指定许多不同的选项，使用诸如where和set这样的关键词。一个典型的执行行有以下形式。

```
sacctmgr modify <entity> set <options> where <options>
```

例子

```
sacctmgr modify user set default=none where default=test
```

将把所有默认账户为 "test "的用户改为账户为 "none"。一旦一个实体被添加、修改或删除，该变化就会被发送到相应的Slurm守护进程，并立即可以使用。

## 移除实体

使用类似于上面的修改例子的执行行来删除实体，但没有设置选项。例如，使用下面的执行行删除所有默认账户为 "test "的用户。

```
sacctmgr remove user where default=test
```

将删除默认账户为 "test "的所有用户记录。

```
sacctmgr remove user brian where account=physic
```

将从 "physics "账户中删除用户 "brian"。如果用户 "brian "有访问其他账户的权限，这些用户记录将会保留。

注意：在大多数情况下，被删除的实体会保留在slurm数据库中，但被标记为删除。如果一个实体存在的时间少于1天，该实体将被完全删除。这是为了清理打字错误。然而，删除用户关联或账户，将导致slurmctld失去对该用户/账户的使用数据的追踪。