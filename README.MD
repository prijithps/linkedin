
---

### **The day I deleted a 5GB log file… and nothing happened**

Early in my DevOps career, a production server was running out of space.
I deleted a **5GB log file**, ran `df -h`, and waited for the disk usage to drop.

It didn’t.
Still **100% full**.
No errors. No warnings. Just confusion.

That’s when I learned one of the most important lessons about Linux file systems:

### **Deleting a file doesn’t always free up space.**

Here’s why ⬇️

In Linux, a “file” has two parts:

* **Filename** → just a pointer
* **Inode** → where the actual data lives

When you delete a file, you remove the **pointer**, not the **data**.
If any process still has that file open, the inode stays alive and keeps consuming disk space.

In my case, the web server was still writing to the **deleted** file.
So the inode remained, invisible in the directory listing but still eating 5GB.

Only after restarting the service — which closed the file handle — did the space finally free up.

---

### **How to troubleshoot this situation**

**1. Check filesystem usage**

```bash
df -h
```

**2. Check directory sizes**

```bash
du -sh /var/log/*
```

**3. Find deleted files still held open by processes**

```bash
lsof +L1
```

If `df` is full but `du` doesn’t add up, this is almost always the reason.

---

### **Bonus: Why logrotate doesn’t just delete files**

Proper log rotation works by **renaming** old files and signaling the process to reopen new ones.
This cleanly closes file handles and avoids exactly this problem.

---

### **Key Takeaways**

* Filenames are just references to inodes
* Space is freed only when no process holds the inode open
* When `df` and `du` disagree, check for open deleted files

Small concept, big impact — and it can save you from some very confusing production incidents.

---

If you want, I can also create a shorter version or add a graphic/diagram for extra engagement!


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
