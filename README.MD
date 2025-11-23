Early in my DevOps career, I deleted a 5GB log file from a production server that was running out of space. 

I ran df -h expecting to see the disk usage drop. It didn’t. 
Still showed 100% full.

No errors, no warnings. Just the same disk usage as before I deleted anything.

That’s when I learned that deleting a file doesn’t always free up space immediately.

In Linux, what we think of as a “file” is actually two separate things: the filename (which is just a pointer) and the inode (which contains the actual data and metadata). When you delete a filename, you’re only removing the pointer. The inode and its data remain on disk as long as any process still has the file open.

In my case, the web server was still writing to that log file. Even though I had deleted the filename, the server process kept its file handle open. The inode stayed alive, invisible to normal file listings but still consuming disk space.

The space was only freed when I restarted the web server, which closed all its file handles.

This is why you need different commands to see the full picture:

# Check filesystem usage
- df -h

# Check actual directory sizes  
- du -sh /var/log/*

# Find deleted files still open by processes
- lsof +L1

The du command shows you what’s actually using space in directories, while df shows filesystem-level usage. 

When they don’t match, you often have deleted files still held open by running processes.

This is also why proper log rotation doesn’t just delete files. Tools like logrotate rename files and send signals to processes so they can close and reopen their file handles cleanly.

Three key takeaways:

1. Filenames are just pointers to inodes
1. Deletion only happens when no processes reference the inode
1. Always check both df and du when troubleshooting disk space

It’s a small detail, but understanding it can save you from confusing production incidents.


You can **easily recreate this scenario** on any Linux machine (even a VM or Docker container).
Here’s a clean, safe demo that shows **how deleting a file doesn’t free space when a process keeps it open**.

---

## ✅ **Step-by-step: Recreate the “deleted file still consuming space” issue**

### **1️⃣ Create a big dummy file**

This simulates a large log file.

```bash
fallocate -l 1G bigfile.log
```

### **2️⃣ Open the file with a process that keeps it “in use”**

Use `tail -f`, which keeps a file handle open indefinitely:

```bash
tail -f bigfile.log &
```

Keep note of the PID (printed automatically or find it with `ps`).

### **3️⃣ Check initial disk usage**

```bash
df -h .
```

### **4️⃣ Delete the file**

```bash
rm bigfile.log
```

Now run:

```bash
ls -lh
```

→ You won’t see the file anymore.

### **5️⃣ Check disk usage again**

```bash
df -h .
```

→ **Disk usage will NOT reduce**, even though the file is gone from the directory.

### **6️⃣ Inspect open deleted files**

```bash
lsof +L1
```

You’ll see something like:

```
tail   1234 user   3r   REG  8,1 1073741824 123456 /path/bigfile.log (deleted)
```

This means the inode still exists because `tail` is holding it.

### **7️⃣ Kill the process**

```bash
kill <PID>
```

### **8️⃣ Check disk usage again**

```bash
df -h .
```

→ Disk usage finally drops, because the inode is released.

---

## 🔍 What just happened?

* You deleted the filename.
* But the **inode remained**, because `tail -f` kept the file handle open.
* The data stayed on disk until the process exited.

This recreates exactly what happened on your production server.

---
