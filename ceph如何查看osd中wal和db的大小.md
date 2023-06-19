---
title: ceph如何查看osd中wal和db的大小
description: 
published: true
date: 2023-06-19T05:54:46.123Z
tags: ceph
editor: markdown
dateCreated: 2023-06-19T05:54:46.123Z
---

您可以使用` ceph daemon osd.ID perf dump `命令来检查 WAL/DB 分区是否即将填满及溢出。`slow_used_bytes` 值显示即将溢出的数据量：

```bash
[ceph: root@storage01 /]# ceph daemon osd.5 perf dump | jq '.bluefs'
{
  "db_total_bytes": 80014729216,
  "db_used_bytes": 52428800,
  "wal_total_bytes": 8589930496,
  "wal_used_bytes": 32505856,
  "slow_total_bytes": 10000827154432,
  "slow_used_bytes": 0,
  "num_files": 23,
  "log_bytes": 3768320,
  "log_compactions": 0,
  "logged_bytes": 1662976,
  "files_written_wal": 1,
  "files_written_sst": 7,
  "bytes_written_wal": 27230208,
  "bytes_written_sst": 86016,
  "bytes_written_slow": 0,
  "max_bytes_wal": 32505856,
  "max_bytes_db": 58720256,
  "max_bytes_slow": 0,
  "alloc_unit_main": 65536,
  "alloc_unit_db": 1048576,
  "alloc_unit_wal": 1048576,
  "read_random_count": 877,
  "read_random_bytes": 3950985,
  "read_random_disk_count": 139,
  "read_random_disk_bytes": 652878,
  "read_random_disk_bytes_wal": 0,
  "read_random_disk_bytes_db": 652878,
  "read_random_disk_bytes_slow": 0,
  "read_random_buffer_count": 738,
  "read_random_buffer_bytes": 3298107,
  "read_count": 664,
  "read_bytes": 26111724,
  "read_disk_count": 75,
  "read_disk_bytes": 26988544,
  "read_disk_bytes_wal": 22654976,
  "read_disk_bytes_db": 4337664,
  "read_disk_bytes_slow": 0,
  "read_prefetch_count": 50,
  "read_prefetch_bytes": 4446042,
  "compact_lat": {
    "avgcount": 0,
    "sum": 0E-9,
    "avgtime": 0E-9
  },
  "compact_lock_lat": {
    "avgcount": 0,
    "sum": 0E-9,
    "avgtime": 0E-9
  },
  "alloc_slow_fallback": 0,
  "alloc_slow_size_fallback": 0,
  "read_zeros_candidate": 0,
  "read_zeros_errors": 0
}
```

