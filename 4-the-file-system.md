# Part 4: The File System

You're logged in! You are now sitting on the login node (`tramuntana`). 

If you type `ls /` (which means "list everything in the root folder"), you will see two very important folders: **`home`** and **`data`**.

---
## 1. Home and Data folders

When you go inside the `/home` directory, you'll have your own folder in there (`/home/your-username`). This is your personal space. It has a specific size limit (quota) assigned to you, typically 200GB.

If you work in a big research group, the data you need for your computations is probably much bigger than 200GB. That's what the `/data` folder is for. It is the massive shared warehouse for your group's large datasets. Inside `/data` you will have your own research group's folder for example if you work in a research group called a-combs, you will have a folder `/data/a-combs`. Inside this folder you can store all your large datasets.

So in `/home` you will only store your codes, scripts and small data files that you personally need to run your jobs. In `/data` you will store all your group's large datasets. 

But these folders (`/data` and `/home`) aren't really on the login node. They actually live on **completely different nodes** — the storage nodes we met in [Part 1](1-under-the-hood.md):

But when you open the `/home` folder on your login node, you are actually looking directly at the hard drives over on the storage nodes. In real time the files are transferred to you over a cable and you read or see them. And each time you want to see them the data has to be fetched again ( if it is not in the memory or RAM ). So its not file sync its data streaming.


---

## 2. The Cable Problem

Now you know that your files live on storage nodes, but your computations happen on compute nodes (like `ada` or `thor`). 

This brings us to a crucial point: **Every time a compute node needs to read or write a file in `/home` or `/data`, that data has to physically travel through a cable.**

Imagine you submit a job to run on `ada`. Your data is sitting in `/home` (which is physically on `migjorn`). Every single time the CPU on `ada` needs a piece of that data, it has to ask `migjorn` for it, the data travels through the cable, arrives at `ada`, gets processed, and if the result needs to be saved, it travels all the way back through the cable to `migjorn`.

If your program reads a file once, does math for an hour, and writes one result file, this is totally fine. But what if your program reads and writes thousands of tiny files every second? 

This is called an **I/O heavy task** (Input/Output heavy. Here reading is input and writing is output). If you do this over the network cable, your job will be incredibly slow, because traveling through the cable takes time. It can also slow down the network for everyone else on the cluster!

---
## 3. Where Should I Put My Data?

Because of how this all works, there are two main ways to handle data when you run jobs, depending on what your job does:

### The "Universal Boss" Strategy (For Normal Jobs)
This is the default setup we just described. There is a Single Source of Truth (the storage nodes). It is made sure that the storage node overrides the `/data` and `/home` paths on every single compute node. 

The beauty of this is that you can write a script on the login node, send it to `ada`, and send it to `thor`, and you **never have to change the file paths**. `/data/my-project` looks exactly the same on every single machine because they are all looking through the portal at the same Boss machine. 

Files are streamed in real-time from the Boss machine when requested. This ensures the entire cluster is looking at the exact same set of files.

### The "Local Storage" Strategy (🚧 Work in Progress)
If your job is reading and writing gigabytes of data constantly (I/O heavy), streaming it over the cable, the cable is a bad idea. 

Instead, it makes much more sense to keep that data physically on the **same computational node** where your job is running. Remember in Part 1 how `ada` has 7TB of NVMe scratch storage and `pampero` has 40TB of local storage? That's what it's for! 

**However, please note that this local storage strategy is not completely implemented right now, so please do not try to use it just yet.** 

Here is how it will work in the near future:
We are planning to take the data folders from these compute nodes and "mount" them directly onto the login node itself. This means you won't need to actually SSH into the compute nodes to put your data there; you'll be able to transfer your data straight from the login node!

We will also partition the compute node's local storage into two parts:
1. **User Data:** One part for you to safely save your data for your tasks.
2. **Internal Buffer:** Another part for the internal buffer of the compute node itself. (Don't worry about what an internal buffer is—it's complicated plumbing and you don't need to know about it!)

So, for now, just know that trying to use this local storage method won't work while we are still implementing it. Stick to the "Universal Boss" strategy!

---

## Recap: What You've Learned

1. **Your files aren't on the login node.** `/home` and `/data` actually live on massive storage servers (`migjorn` and `tramuntana-nas`).
2. **The Data Cable Problem:** Accessing shared storage from a compute node means data has to travel over a cable. 
3. **Universal Boss vs Local Storage:** For most jobs, reading from the shared storage (the Boss) is perfect. For I/O heavy jobs, it's better to copy the data to the compute node's local hard drive first so it doesn't have to travel through the cables constantly.

---

**Next up**: In [Part 5: Your First Job](5-your-first-job.md), we'll finally put everything together and you'll write and submit your very first batch job to the cluster using SLURM.

---
**Navigation:**
🏠 [Home](README.md) | 📖 [1. Under the Hood](1-under-the-hood.md) | ⚙️ [2. What is SLURM?](2-what-is-slurm.md) | 🔌 [3. Getting Connected](3-getting-connected.md) | 📁 **4. The File System** |
---
