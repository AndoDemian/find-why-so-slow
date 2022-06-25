# Find - Why so slow
### _Date published: 2019-12-11_

---

*Disclaimer: Assumption is the mother of all f\*ck ups. Test on your own.*

## Setup

For me to notice a difference in speed, I had to bring back to life an old machine.
```bash
    $ fio --randrepeat=1 --ioengine=libaio --direct=1 --gtod_reduce=1 --name=test --filename=test --bs=4k --iodepth=64 --size=4G --readwrite=randrw --rwmixread=75
    
    test: Laying out IO file (1 file / 4096MiB)
    Jobs: 1 (f=1): [m(1)][100.0%][r=800KiB/s,w=216KiB/s][r=200,w=54 IOPS][eta 00m:00s]   
    test: (groupid=0, jobs=1): err= 0: pid=6059: Mon Dec  9 19:28:53 2019
       read: IOPS=115, BW=461KiB/s (472kB/s)(3070MiB/6821096msec)
       bw (  KiB/s): min=    8, max= 1480, per=100.00%, avg=461.36, stdev=134.18, samples=13633
       iops        : min=    2, max=  370, avg=115.27, stdev=33.55, samples=13633
      write: IOPS=38, BW=154KiB/s (158kB/s)(1026MiB/6821096msec)
       bw (  KiB/s): min=    7, max=  512, per=100.00%, avg=154.63, stdev=56.57, samples=13589
       iops        : min=    1, max=  128, avg=38.59, stdev=14.14, samples=13589
      cpu          : usr=0.09%, sys=0.56%, ctx=1032374, majf=0, minf=8
      IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
         submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
         complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
         issued rwt: total=785920,262656,0, short=0,0,0, dropped=0,0,0
         latency   : target=0, window=0, percentile=100.00%, depth=64
    
    Run status group 0 (all jobs):
       READ: bw=461KiB/s (472kB/s), 461KiB/s-461KiB/s (472kB/s-472kB/s), io=3070MiB (3219MB), run=6821096-6821096msec
      WRITE: bw=154KiB/s (158kB/s), 154KiB/s-154KiB/s (158kB/s-158kB/s), io=1026MiB (1076MB), run=6821096-6821096msec
    
    Disk stats (read/write):
      sda: ios=783667/271964, merge=5235/4130, ticks=340948708/107532860, in_queue=448661056, util=100.00%
```    

---

## Scope

- 222773 - dirs; 964760 - files; 16 - max depth;
- The file that I was searching for was on the 4th level.
- After each test, I've cleared pagecache, dentries, and inodes.  
    `echo 3 | sudo tee /proc/sys/vm/drop_caches`

---    

## Findings

- Baseline
```bash
    find files/ -name "test_file"
    real    9m13,494s
    user     0m5,095s
    sys     0m18,677s
```    

Now let's see how we can optimize it; because this is terrible.



- Optimization method: delimit the number of traversing directories.
```bash
    find files/ -maxdepth 4 -name "test_file" 
    real    1m29,955s
    user     0m0,398s
    sys      0m2,135s
```    

Ok, we are getting somewhere. Let's keep it up!

</br>

- Optimization method: We know exactly the minimum and maximum depth.
```bash
    find files/ -mindepth 4 -maxdepth 4 -name "test_file"  
    real    1m28,972s
    user     0m0,360s
    sys      0m2,250s
```

Interesting. It will traverse all the directories to establish the minimum tree depth. That doesn't help us.

</br>

- Optimization method: Trow a bunch of arguments, surely that will optimize the execution plan.
```bash
    find files/ -maxdepth 4 -size +100M -user lnxusr -perm 0644 -type f -name "test_file" 
    real    1m28,556s
    user     0m0,481s
    sys      0m2,112s
```    

Well ... that didn't go as planned.

</br>

- Optimization method: Maximum performance!!!
```bash
    ionice -c1 -n0 nice -n -20 find files/ -name "test_file"
    real    10m31,983s
    user      0m5,155s
    sys      0m20,752s
```    

What? How? Why?

</br>

Let's dig deeper.  
***Kowalski, Analysis!***

I have extracted the syscalls and how many times they are called with strace.  

Looking at it it's starting to make sense, more args = more system calls to evaluate if the file should be printed to the user or not.  

</br>

The reason I've said cache is the magic is that after you running find, the performance will considerably improve.  

- Optimization method: Cached files
```bash
    find files/ -maxdepth 4 -name "test_file"
    real    0m0,559s
    user    0m0,246s
    sys     0m0,311s
```

Every time a file is read, Linux will check the Memory to see if it's there, if it's not, it will store it for future lookups.

---

## Conclusion

- Start the find from a path closer to the file you want to search for.
    - Don't start from / (root) and expect performance.

</br>

- Maxdepth it's the only thing that will help you, especially when you have big tree structures.
    - Start with a low value and change your starting path as you progress.

</br>

- Nice, ionice don't help.
    - I've cranked everything to Real-time, Highest Performance, nothing. Don't use it.

