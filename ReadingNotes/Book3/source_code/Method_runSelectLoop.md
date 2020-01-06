* Android 8.0
* [源码地址：frameworks/base/core/java/com/android/internal/os/ZygoteServer.java](https://www.androidos.net.cn/android/8.0.0_r4/xref/frameworks/base/core/java/com/android/internal/os/ZygoteServer.java)

```java
    /**
     * Runs the zygote process's select loop. Accepts new connections as
     * they happen, and reads commands from connections one spawn-request's
     * worth at a time.
     *
     * @throws Zygote.MethodAndArgsCaller in a child process when a main()
     * should be executed.
     */
    void runSelectLoop(String abiList) throws Zygote.MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        fds.add(mServerSocket.getFileDescriptor()); // 1
        peers.add(null);
      
        // 无限循环等待AMS的请求
        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) { // 2
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) { // 3
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                if (i == 0) { // Zygoye进程和AMS建立了连接
                    ZygoteConnection newPeer = acceptCommandPeer(abiList); // 4 
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else { // 创建新的进程
                    boolean done = peers.get(i).runOnce(this); // 5
                    if (done) {
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    }
```

* 注释1处的`mServerSocke`t就是我们在`registerZygoteSocket`函数中创建的服务器端Socket，调用`mServerSocket.getFileDescriptor()`函数来获取该Socket的fd字段的值并添加到fd列表`fds`中。接下来无线循环用来等待AMS请求Zygote进程创建新的应用程序进程。
* 在注释2处通过遍历将`fds`存储的信息转移到`pollFds`数组中。
* 在注释3处对`pollFds`进行遍历，如果`i==0`成立，说明服务器端Socket于客户端连接上了，换句话说就是，当前Zygote进程与AMS建立了连接。
* 在注释4处通过`acceptCommandPeer`方法得到`ZygoteConnection`类并添加到Socket连接列表`peers`中，接着将改`ZygoteoConnection`的fd添加到fd列表`fds`中，以便可以接收到AMS发送过来的请求。如果`i`的值不等于0，则说明AMS向Zygote进程发送了一个创建应用进程的请求。
* 则在注释5处调用`ZygoteConnection`的`runOnce`函数来创建一个新的应用程序进程，并在成功创建后将这个连接从Socket列表`peers`和fd列表`fds`中清除。