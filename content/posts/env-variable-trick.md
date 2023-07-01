---
title: "Environment variable trick on Linux"
date: 2021-07-24T14:44:55+03:00
draft: false
---

Hi, In this post I’m going to show you a trick which can be useful in some circumstances. Okay, let me make it more clear with a scenario. Consider following scenario:

Assume that you want to run a shell script on Linux. And you don’t have permission to modify this script file. There might be some check(e.g. hash) to avoid modification. And shell script is like following:

```bash
#!/bin/bash

# YOU SHALL NOT CHANGE THIS FILE!

java=$(which java)

$java -cp /path/to/classes MySimpleApp
```

It is very trivial script, right? Using which command, we looked the PATH environment variable for where java executable is placed. So, what is the problem here? Or, is there any problem here?↳

Actually, it depends. Might the answer related to Java versions? What if you have to execute the above shell script as root user and with Java 11. But wait, Java 8 is default for root user. Hmm, it is trivial, I can change the root user’s PATH variable .bashrc file, right? Yes, you can BUT please DON’T. Because there could be running services rely on Java 8. And if you alter the PATH variable in the .bashrc file, every services and shell sessions(as root user) will be affected. We don’t want that, we don’t want to break existing things in the system.
Well, you may say “You’ve made me upset! What can I do?”. Okay the trick is altering the PATH variable temporarily. We can give the environment variables while executing sudo command. Like following:

```bash
sudo PATH="/opt/java11/bin:$PATH" bash my_shell_script.sh
```

That’s it! Now, it is running with Java 11 temporarily. You didn’t alter any environment variable permanently.

The above one is first approach and there is an alternative:

```bash
su - root

export PATH="/opt/java11/bin:$PATH" 

bash my_shell_script.sh
```

First, open a login shell with “su -“ as root user. Then export a environment variable like we did in sudo command. This change is also temporary and specific to shell session and it expires when the shell session is closed. Then finally, execute the target shell script.

This is the end of the post, hope you enjoyed! Please feel free to contact me if you have any suggestions, questions, concerns.