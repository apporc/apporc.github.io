systemd 对你的进程数限制横插一脚啦
==

以往在执行海量进程并发的时候，看到这错："fork:retry:no child process"，
通常就想到 ulimit 了。不过现在要注意，它也可能是 systemd 的梗喽。

#### ulimit

*   回顾一下 ulimit

    内核对于用户可占用的资源可以作一些限制，ulimit 是调整这些限制的一个接口。
    这个接口就是个命令而已，主要用于系统运行时临时调整一些资源的限制，
    ulimit 命令是 bash 的内建命令，注意它是用于设置当前 shell 以及其子进程中
    的资源额度的。其它 shell 是怎么支持这个命令的这里暂略；

    对应的，如果想要永久性地调整资源限制，则需要更改文件
    `/etc/security/limits.conf` 或 `/etc/security/limits.d`
    我查看了一下，在我的 archlinux 以及 centos 7 上这个目录都是一致的，猜测
    应该不会有发行版自定义这个位置，这些配置文件是 pam_modules.so 内核模块的。

*   下面来摸一下它们

    *   执行 `ulimit -a` 查看一下所有可设置的项目

            # ulimit -a
            core file size          (blocks, -c) 0
            data seg size           (kbytes, -d) unlimited
            scheduling priority             (-e) 0
            file size               (blocks, -f) unlimited
            pending signals                 (-i) 31858
            max locked memory       (kbytes, -l) 64
            max memory size         (kbytes, -m) unlimited
            open files                      (-n) 1024
            pipe size            (512 bytes, -p) 8
            POSIX message queues     (bytes, -q) 819200
            real-time priority              (-r) 0
            stack size              (kbytes, -s) 8192
            cpu time               (seconds, -t) unlimited
            max user processes              (-u) 4096
            virtual memory          (kbytes, -v) unlimited
            file locks                      (-x) unlimited

        基本上，每项的意义从名称中已知大概。本文关心的主要是 max user processes
        这一项，用户最大进程数，超过这个进程数就 fork 不了啦。

    *   执行 `ulimit -u` 来设置最大进程数

            # ulimit -u 8192
        设置过后，再用 `ulimit -a` 就可以看到效果啦。
        如果你看到这个 "Operation not permitted"，是因为作为普通用户权限不够。
        如果你无法获得 root 权限，那么 `ulimit -S` 是你的选择，`-S` 是说设置
        soft limit，用于普通用户设置自己的子进程资源限制， `-H` 对应的就是
        hard limit，`ulimit` 默认使用 `-H` 选项，只有 root 有权限使用。

    *   查看 `/etc/security/limits.conf`

        有以下四栏

            <domain>      <type>  <item>         <value>
        每一栏的含义，在默认配置文件中已经有解释，以下以 test 用户为例，
        设置其最大进程数的 soft limit 与 hard limit 分别为 1024 和 8192

            @test		soft	noproc          1024
            @test		hard 	noproc  		8192

        下面是文件中对于资源项的解释，比 ulimit 给出的详细一点：

            core - limits the core file size (KB)
            data - max data size (KB)
            fsize - maximum filesize (KB)
            memlock - max locked-in-memory address space (KB)
            nofile - max number of open file descriptors
            rss - max resident set size (KB)
            stack - max stack size (KB)
            cpu - max CPU time (MIN)
            nproc - max number of processes
            as - address space limit (KB)
            maxlogins - max number of logins for this user
            maxsyslogins - max number of logins on the system
            priority - the priority to run user process with
            locks - max number of file locks the user can hold
            sigpending - max number of pending signals
            msgqueue - max memory used by POSIX message queues (bytes)
            nice - max nice priority allowed to raise to values: [-20, 19]
            rtprio - max realtime priority

    *   查看 `/etc/security/limits.d` 目录

        目录下是一些子配置文件，其配置项定义格式与 `/etc/security/limits.conf`
        中相同，使用场景是将不同程序的配置放在目录下作为独立文件，便于管理。

#### systemd TasksMax
systemd 也提供了一个基于 cgroup 的限制资源使用的机制。
最早宣布[在此][1]，代码提交[在此][2]，默认值设置[在此][3]，stackoverflow 上
有一个与此相关的[提问][4]。

*   TasksMax 
    对于任一 systemd 服务来说，在其服务文件中，设置 TasksMax 值来限制最大进程数。
    官方术语是说设置最大任务数，落实到 cgroup 中就是 pids.max 值，所以我认为说是
    进程数应该是可以的。 systemd 服务的所有属性，参见官方文档[解释][5]。

    以我的 tmux@.service 文件来说，在 `[Service]` 下设置 TasksMax 即可

        [Service]
        Type=forking
        User=%I
        Restart=always
        TasksMax=infinity # infinity 是指不限制
        ExecStart=/usr/bin/tmux new-session -s %I -n %I -c /home/%I/ -d
        ExecStop=/usr/bin/tmux kill-server

    systemd service 文件的写法，不在本文范围内。

*   DefaultTasksMax
    TasksMax 的默认值由 DefaultTasksMax 指定。
    在文件 `/etc/systemd/system.conf` 或者 `/etc/systemd/user.conf` 都可以设置，
    分别掌管系统级服务与用户级服务的默认值。

    该值默认为 512。

*   UserTasksMax
    除了以上两个值之外，还有 UserTasksMax 值，该值限定了从一个 login-shell
    中可以运行的任务数的限制。在 `/etc/systemd/logind.conf` 的 [login] 段下设置。

    默认为 4096。

 [1]: https://github.com/systemd/systemd/blob/master/NEWS#L60
 [2]: https://github.com/systemd/systemd/pull/1239
 [3]: https://github.com/systemd/systemd/pull/1886
 [4]: http://unix.stackexchange.com/questions/253903/creating-threads-fails-with-resource-temporarily-unavailable-with-4-3-kernel
 [5]: https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#TasksMax=N
