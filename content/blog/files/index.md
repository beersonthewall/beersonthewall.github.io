---
title: "Title"
date: 2021-01-02T10:06:46-06:00
draft: true
description: "Two or three sentence description of the post."
tags: []
---
Quick note to self, all this is essentially verbatium from: https://www.deconstructconf.com/2019/dan-luu-files
Not originaly in any way. DONT PUBLISH, these are just notes for the post.

# Files are fraught with peril:

## API

The goal: take a file with "a foo" and modify it to "a bar".

```
pwrite(
    [file],
    "bar", // Data to write
    3, // write 3 bytes
    2 // at offset 2
);
```

  So maybe this works, but we could crash. Either before writing or in the middle of writing.
  pwrite not atomic on crash.
  Standard technique: 
  1) copy existing file to undo log
  2) modify file (recover on crash)
  3) delete log file
  
  ```
  creat(/d/log)
  write(/d/log, "2,3,foo", 7)
  pwrite(/d/orig, "bar", 3, 2)
  unlink(/d/log)
  ```
  
  This may or may not work. ext3, data=journal works.
  
  data=ordered can re-order our write operations so we need to fysnc(/dir/myfile) which is a barrier (prevents
  re-ordering) and flushes caches.
  
  Sometimes you'll go to restore from crash and won't be able to see the file, so you gotta fsync
  the directory metadata too. This gives you:
  
    ```
  creat(/d/log)
  write(/d/log, "2,3,foo", 7) //data=writeback, write not necessarily atomic. file could
                              //be re-sized and we see random data. Add a check sum.
  fsync(/d/log)
  fsync(/d)
  pwrite(/d/orig, "bar", 3, 2)
  fsync(/d/orig)
  unlink(/d/log)
  ```
  fysnc afterwords to make sure all our work is saved. Also each of those files can return errors.
  Checking those is removed for brevity.

  Most programs written by expects (databases, version control, etc) have bugs 
  (this stuff is capital H hard).
  
  Common problems:
  1) Incorrectly assume operations are atomic (e.g. assuming pwrite is atomic on crash)
  2) Assuming operations execute in-order (re-ordering issues with write / pwrite)
  3) Infrequent non-deterministic errors
  4) Concurrency is hard
  5) API is inconsistent (by design)
     stuff is atomic conditionally on data modes and also re-ordered conditionally.
     every property you care about has conditions attached.
     example: write a single sector to disk atomically. Not guranteed by filesystems, just happens
     to be done by most disks. New disks (NVM non volatile memory) may not guarantee this.
     TLDR code exactly to POSIX spec?
     Also probably read last 5-10 years of LKML wrt filesystems
     
## Filesystem
    Sort of handle simple errors. But there are many cases where errors will go unreported.
    Error on fsync:
    Linux used to report the error to the wrong process. Today it's most likely reported correctly,
    but essentially unrecoverable.

## Disks
    Lie about power safety.
    Disks often don't flush data (Rajimwale et al, DNS'11)
    Error rate is orders of magnitude higher than reported by vendors.
    SSDs need ECC to function at all, does not fix any error rate issues.
    Also SSDs are pretty "forgetful"
