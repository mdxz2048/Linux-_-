# 1-signal

##  实验

1. 编译程序

    ```sh
    gcc signal.c -o signal
    ```

    

2. 运行`signal`程序，可以看到当前进程的PID为**19831**(PID与当前实验环境有关，不一定每次都一样);

    ```sh
    $ ./signal
    The pid is 19831
    ```

    

3. 复制ssh会话，在另一个窗口向进程19831发送SIGIO信号；

    ```sh
    kill -SIGIO 19831
    ```

4. 可以看到signal进程收到了这个信号，执行了对应的信号处理函数；

    ```sh
    $ ./signal
    The pid is 19831
    received SIGIO
    ```

5. 这里，可以试着去去掉程序中`signal(SIGIO, sig_handler)` 这一行的挂接函数，再编译执行，同样发送SIGIO信号，可以看到进程执行了信号的默认处理，被直接退出；

    ```sh
    The pid is 20195
    [1]    20195 pollable event occurred  ./signal
    ```

    

