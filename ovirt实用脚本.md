---
title: ovirt实用脚本
description: 翻译自：https://www.ovirt.org/develop/developer-guide/db-issues/helperutilities.html#taskcleaner
published: true
date: 2023-06-05T14:10:39.815Z
tags: kvm, ovirt
editor: markdown
dateCreated: 2023-06-05T13:48:54.754Z
---



翻译自：[HelperUtilities | oVirt](https://www.ovirt.org/develop/developer-guide/db-issues/helperutilities.html#taskcleaner)

目录/usr/share/ovirt-engine/setup/dbutils或在开发者设置中的$PREFIX/share/ovirt-engine/setup/dbutils包含有用的脚本，帮助解决各种数据库问题。

## fkvalidator

`fkvalidaor.sh`是一个在升级前必须在客户数据库上运行的脚本，该工具检查数据库内的所有数据是否一致，并且没有破坏任何FK约束 fkvalidaor.sh列出了发现的问题，并且在使用-f开关时可以修复这些问题

Usage: `fkvalidator.sh [-h] [-s SERVERNAME [-p PORT]] [-d DATABASE] [-u USERNAME] [-l LOGFILE] [-f] [-v]`

```
-s SERVERNAME - The database servername for the database  (def. localhost)
-p PORT       - The database port for the database        (def. 5432)
-d DATABASE   - The database name                         (def. engine)
-u USERNAME   - The admin username for the database.
-l LOGFILE    - The logfile for capturing output          (def. fkvalidator.sh.log)
-f            - Fix the non consistent data by removing it from DB.
-v            - Turn on verbosity                         (WARNING: lots of output)
-h            - This help text.
```

## 任务清理器

`taskcleaner.sh`是一个用于清理异步任务和相关工作步骤/补偿数据的工具。该工具可以： 1： `Display`

```
   All async tasks
   Only Zombie tasks
```

删除

```
All tasks
All Zombie tasks
A task related to a given task id
A Zombie task related to a given task id
All tasks related to a given command id
All Zombie tasks related to a given command id
```

可以添加标志(-C, -J)来指定是否也要清理工作步骤和报酬数据。

### 用法

```
Usage: taskcleaner.sh [-h] [-s server] [-p PORT]] [-d DATABASE] [-u USERNAME] [-l LOGFILE] [-t taskId] [-c commandId][-z] [-R] [-C][-J] [-q] [-v]
-s SERVERNAME - The database servername for the database (def.localhost)
-p PORT - The database port for the database (def. 5432)
-d DATABASE - The database name (def.engine)
-u USERNAME - The admin username for the database.
-l LOGFILE - The logfile for capturing output (def.taskcleaner.sh.log)
-t TASK_ID - Removes a task by its Task ID.
-c COMMAND_ID - Removes all tasks related to the given Command Id.
-z - Removes/Displays a Zombie task.
-R - Removes all Zombie tasks.
-C - Clear related compensation entries.
-J - Clear related Job Steps.
-q - Quite mode, do not prompt for confirmation.
-v - Turn on verbosity (WARNING: lots of output)
-h - Help text.
```

## unlock_entity

unlock_entity.sh是一个用于解锁虚拟机、模板和/或其相关磁盘或特定磁盘的工具。虚拟机、模板是由其名称提供的，而特定磁盘则由其UUID提供。

### 使用方法

```
Usage: ./unlock_entity.sh [options] [ENTITIES]
-h            - This help text.
-v            - Turn on verbosity                         (WARNING: -l LOGFILE    - The logfile for capturing output          (def. )
-s HOST       - The database servername for the database  (def. localhost)
-p PORT       - The database port for the database        (def. 5432)
-u USER       - The username for the database             (def. engine)
-d DATABASE   - The database name                         (def. engine)
-t TYPE       - The object type {vm | template | disk | snapshot}
-r            - Recursive, unlocks all disks under the selected vm/template.
-q            - Query db and display a list of the locked entites.
ENTITIES      - The list of object names in case of vm/template, UUIDs in case of a disk
NOTE: This utility access the database and should have the
corresponding credentals.
In case that a password is used to access the database PGPASSWORD
or PGPASSFILE should be set.
Example:
$ PGPASSWORD=xxxxxx ./unlock_entity.sh -t disk -q
```