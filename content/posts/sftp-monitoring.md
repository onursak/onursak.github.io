---
title: "Monitoring download/upload progress using JSch library"
date: 2021-07-06T14:44:55+03:00
draft: false
---

In this post, I’m going to show you how to monitor progress of download/upload using JSch library in Java. Let’s see how to do it step by step:

### 1. Establishing SSH connection

Before using SFTP, first we need to establish SSH connection to the remote host. Following code demonstrates that how we can do this using JSch. (Of course, following codes don’t have any concern of following good coding practices. So, when you implement your code, please consider making your methods more modular, parameterized etc.)

```java
    public Session connectSession() throws JSchException {
        JSch jsch = new JSch();
        Properties config = new Properties();
        // Disable StrictHostKeyChecking to directly connect using username and
        // password, it's not a good practice for security concerns, not
        // recommended when you are not working in internal network etc.
        config.put("StrictHostKeyChecking", "no");
        Session session = jsch.getSession("username", "host", 22);
        session.setConfig(config);
        session.setPassword("password");
        session.connect();
        return session;
    }
```
    

### 2. Opening SFTP channel using session

Now, we have SSH session and we can open a SFTP channel. It’s very simple and can be seen from following code:


```java
    public ChannelSftp connectSftpChannel(Session session) throws JSchException {
        ChannelSftp sftpChannel = (ChannelSftp) session.openChannel("sftp");
        sftpChannel.connect();
        return sftpChannel;
    }
```


### 3. Implementing our SftpProgressMonitor class

If we look at ChannelSftp(provided by JSch library) class’ source code, it provides bunch of overloaded **get(…)** and **put(…)** methods.

**get(…)** method is used to download a file, and **put(…)** method is used to upload a file.

Let’s look at following method signatures from ChannelSftp class:

```java
    public void get(String src, String dst, SftpProgressMonitor monitor, int mode) throws SftpException;
    
    public void put(String src, String dst, SftpProgressMonitor monitor, int mode) throws SftpException;
```

(For rest of the post, when I mention get(…) or put(…), I’ll mean the above method signatures)

I think you noticed that there is a parameter of type SftpProgressMonitor. SftpProgressMonitor is an interface provided by JSch. Library says that “Hey, give me your implementation of this interface and I will use its methods inside get(…) and put(…) methods.” So, let’s look at SftpProgressMonitor interface:

```java
    public interface SftpProgressMonitor{
        public static final int PUT=0;
        public static final int GET=1;
        public static final long UNKNOWN_SIZE = -1L;
        void init(int op, String src, String dest, long max);
        boolean count(long count);
        void end();
    }
```
 
So, you need to implement init, count and end methods in your own implementation. But what do these methods mean actually? When these methods are executed?

For example, assume that you want to download a 100 MB file from remote host and you run get(…) method.

Assume that we called the get(…) method like this:

```java
    sftpChannel.get("/source/test.txt", "/target/test.txt", new MySftpProgressMonitor(), ChannelSftp.RESUME);
```

Let’s trace what’s happening in methods by simplifying: (You can think this as debugging.)

```java
    // When we call get(...) method with above parameters, monitor object's
    // methods will be called like this:
    
    // init method will be called at the beginning of the get(...) method
    monitor.init(SftpProgressMonitor.GET, "/source/test.txt", "/target/test.txt", 104857600); // 104857600 is the size in bytes of the source file which is 100 MB
    
    // count method will be called at each iteration during file transfer, the
    // given parameter is the byte size is transfered in an iteration
    // if the mode is RESUME then get method will check the dest file is already
    // there and if it is, then the count method will be called with already
    // transferred bytes at the beginning, and continues to increment at each
    // iteration
    monitor.count(10000); // 10000 is a fake number, don't focus on it
    
    // end method will be called when the transfer is finished
    monitor.end();
```

Now, we’ve seen a lot of things and we’re ready to implement our SftpProgressMonitor. I hope following implementation gives you an inspiration about how to implement your own:

```java
    public class MySftpProgressMonitor implements SftpProgressMonitor {
        private long totalBytes;
        private long transferredBytes;
        private NumberFormat formatter = new DecimalFormat("#0.00");
    
        public MySftpProgressMonitor(){
    
        }
    
        @Override
        public void init(int op, String src, String dest, long max) {
            totalBytes = max;
            System.out.println("Sftp Progress monitor is initialized for op " + op
                                + " src: " + src + " dest: " + dest + " max: " + max);
        }
    
        @Override
        public boolean count(long count) {
            transferredBytes += count;
            double progress = (double) transferredBytes/totalBytes;
            System.out.println("Current progress is: %" + formatter.format(progress*100));
            return true;
        }
    
        @Override
        public void end() {
            System.out.println("Sftp Progress monitor is ended.");
        }
    }
```

Well, the above code is okay and functional but you may realized that printing in count() method is not the best solution because the count() method can be excessively called in get(…) method. So, the following implemantation would be a better solution:

```java
public class MySftpProgressMonitor implements SftpProgressMonitor {
        private long totalBytes;
        private long transferredBytes;
        private double progress;
    
        public MySftpProgressMonitor(){
    
        }
    
        @Override
        public void init(int op, String src, String dest, long max) {
            totalBytes = max;
        }
    
        @Override
        public boolean count(long count) {
            transferredBytes += count;
            progress = (double) transferredBytes/totalBytes;
            return true;
        }
    
        // Reset the values, so the same MySftpProgressMonitor object can be reused
        // within same thread
        @Override
        public void end() {
            totalBytes = 0;
            transferredBytes = 0;
            progress = 0;
        }
    
        // Provide getters, so you can retrieve the information only when it is
        // needed, now this class is only responsible for tracking information not
        // for printing/logging etc.
        public long getTotalBytes(){
            return totalBytes;
        }
    
        public long getTransferredBytes(){
            return getTransferredBytes;
        }
    
        public double getProgress(){
            return progress;
        }
    }
```

This is the end of the post, hope you enjoyed! Please reach me if you have any questions, suggestions etc.