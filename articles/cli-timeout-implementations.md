---
title: "各種CLIツールのタイムアウト実装まとめ"
emoji: "🔨"
type: "tech"
topics: ["linux"]
published: false
---

本文は [2014年に書いた文章](https://github.com/hiboma/hiboma/blob/master/各種ツールのタイムアウト実装まとめ.md) を zenn 向けに再編集した内容です

## 序論

_タイムアウト_ は難しい

## 検証環境

socket(2) でソケットを扱う各種ツールで、到達できない適当な IP (例では `192.168.100.1` を使った) を指定して ___タイムアウト___ の検証をした。

 * 調べたプロトコルは TCP/IP、一部 UDP
 * クライアント側だけ検証
 * /proc/sys/net/ipv4/tcp_syn_retries はデフォルト値
 * Vagrant の CentOS6.5 2.6.32-431.el6.x86_64

注意) 適当な IP じゃなくても iptables で DROP してテストしてもいいはず

## 比較表

　| option | syscall to timeout | 何のタイムアウト? | 
----|:----:|:----:|:----:
`wget` | `--dns-timeout` | `setitimer(2) + ITIMER_REAL` | ホスト名前解決のタイムアウト (nscd, hosts, dns 等) |
`wget` | `--connect-timeout` | `setitimer(2) + ITTIMER_REAL` | connect(2) のタイムアウト
`wget` | `--read-timeout` | `select(2)` | connect(2)後、idle時間(= select(2)) のタイムアウト
`wget` | `--timeout` |  | 上記三つをまとめてセットする怠惰オプション | 
`curl` | `--connect-timeout` | `O_NONBLOCK + poll(2)` | connect(2) のタイムアウト
`curl` | `--max-time` | `alarm(2)` | curl を実行してからの実時間でのタイムアウト
`nc` |  | `select(2)` | タイムアウト値無し。TCPの再送回数が上限で connect(2) がタイムアウト | 
`nc` | `-w` | `select(2)` | connec(2)がタイムアウト <br /> ESTABLISED な時は idle 時間がになったらタイムアウトぽい
`mysql` | |  `connect(2)` | TCPの再送回数が上限で connect(2) がタイムアウト |
`mysql` | `--connect_timeout` | `O_NONBLOCK + poll(2)` | 認証を通さないとタイムアウト? |
`rsync` | `--timeout`         | `select(2)` | I/O の待機時間のタイムアウト |
`rsync` | `--contimeout`      | `alarm(2)` | rsync daemon に connect(2) するタイムアウト |

___タイムアウト___ といってもコンテキストが多様であることが分かる

 * connect(2) = 3way-handshake のタイムアウト
 * conncet(2) に成功し、その後のデータ転送のタイムアウト
 * ホスト名解決のタイムアウト
 * コマンド実行時間のタイムアウト

これは一例である。コマンドやプロトコルによってもっと細かくタイムアウトが決められている物もある (例: mysql ) オプション指定がどのタイムアウト値を変更しているのか把握しておくと、問題やトラブルを精緻に調べられるでしょう。

TCP の SYN再送も暗にタイムアウトに絡んでくるので把握するとよいでしょう。

# TCP の SYN再送

 * TCP の 3way-handshake で SYN-ACK を受け取らないと発生します

## TCP のSYN再送を見る

カーネルが勝手にやってるので strace じゃ観測できないよ。tcpdump 等で見ておけばよい (2020年なら eBPF 関連のツールも使えるでしょう)

 * 他に何もプロセスいなければ `sudp tcpdump port 80` で見たりする
   * サンプルが多ければ適当に IP 絞るとかしてください。 man tcpdump
 * 再送の間隔については http://d.hatena.ne.jp/rx7/20131129/p1 も大事

## TCP の SYN再送でタイムアウトになるケース

SYN再送の回数を小さくすると再送のタイムアウト値が短くなるため、ツールで指定した数値よりも早くタイムアウトしうる

```
# 不達の IP に繋ごうとしてタイムアウト
$ time nc 192.168.100.1 80

real    1m3.001s
user    0m0.000s
sys     0m0.000s
```

 * nc は select(2) のタイムアウトを指定しないのでずっとブロックするはずだが ...
 * SYNの再送上限に達してタイムアウトする

下記がSYN再送の tcpdump。 デフォルトCentOS6.5 だと 1+5回再送で1分間かかる

```
00:49:58.737282 IP 10.0.2.15.38278 > 192.168.100.1.http: Flags [S], seq 2741378757, win 14600, options [mss 1460,sackOK,TS val 117887789 ecr 0,nop,wscale 7], length 0
00:49:59.736376 IP 10.0.2.15.38278 > 192.168.100.1.http: Flags [S], seq 2741378757, win 14600, options [mss 1460,sackOK,TS val 117888789 ecr 0,nop,wscale 7], length 0
00:50:01.737149 IP 10.0.2.15.38278 > 192.168.100.1.http: Flags [S], seq 2741378757, win 14600, options [mss 1460,sackOK,TS val 117890789 ecr 0,nop,wscale 7], length 0
00:50:05.737388 IP 10.0.2.15.38278 > 192.168.100.1.http: Flags [S], seq 2741378757, win 14600, options [mss 1460,sackOK,TS val 117894790 ecr 0,nop,wscale 7], length 0
00:50:13.738313 IP 10.0.2.15.38278 > 192.168.100.1.http: Flags [S], seq 2741378757, win 14600, options [mss 1460,sackOK,TS val 117902790 ecr 0,nop,wscale 7], length 0
00:50:29.738310 IP 10.0.2.15.38278 > 192.168.100.1.http: Flags [S], seq 2741378757, win 14600, options [mss 1460,sackOK,TS val 117918790 ecr 0,nop,wscale 7], length 0
```

ツールで指定したタイムアウトと合わせて SYN再送の設定も確認する必要があります。

#### SYN再送の回数を減らしてタイムアウトを発生させる

/proc/sys/net/ipv4/tcp_syn_retries を変えてテストしてみよう

```sh
echo 0 > /proc/sys/net/ipv4/tcp_syn_retries 
```

0 にしたので SYN 再送回数が 0 に ... ならなかった

0 にしても必ず一回の再送 ( 1回目の SYN 送信 -> 1秒 待つ -> 2回目の SYN 送信 -> 2秒待つ ) が行われて、実時間で __3秒以上___ かかるのだった

```
# 最初の SYN 送信
01:31:03.101121 IP 10.0.2.15.48866 > 192.168.100.0.http: Flags [S], seq 3817435773, win 14600, options [mss 1460,sackOK,TS val 120352153 ecr 0,nop,wscale 7], length 0
# 1秒後に再送
01:31:04.100374 IP 10.0.2.15.48866 > 192.168.100.0.http: Flags [S], seq 3817435773, win 14600, options [mss 1460,sackOK,TS val 120353153 ecr 0,nop,wscale 7], length 0
```

### curl でSYN再送タイムアウトのテスト

```
$ time curl --connect-timeout 999 192.168.100.0
curl: (7) couldn't connect to host

real    0m3.004s
user    0m0.002s
sys     0m0.001s
```

3秒でタイムアウト。 poll が POLLERR|POLLHUP 返している

```
connect(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("192.168.100.0")}, 16) = -1 EINPROGRESS (Operation now in progress)
poll([{fd=3, events=POLLOUT|POLLWRNORM}], 1, 1000000) = 1 ([{fd=3, revents=POLLERR|POLLHUP}])
getsockopt(3, SOL_SOCKET, SO_ERROR, [8589934702], [4]) = 0
```

### wget でSYN再送タイムアウトのテスト 

```
$ time wget --tries 1 --connect-timeout 999 192.168.100.0
--2014-03-15 00:42:13--  http://192.168.100.0/
Connecting to 192.168.100.0:80... failed: Connection timed out.
Giving up.


real    0m3.003s
user    0m0.001s
sys     0m0.001s
```

3秒でタイムアウト。wget は connect(2) が ETIMEDOUT 返す

```
setitimer(ITIMER_REAL, {it_interval={0, 0}, it_value={999, 0}}, NULL) = 0
connect(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("192.168.100.0")}, 16) = -1 ETIMEDOUT (Connection timed out)
```

# 各種ツールの挙動を調べる

# curl --connect-timeout

```
curl --connect-timeout 5 192.168.100.1
```

## USAGE

```
       --connect-timeout <seconds>
              Maximum  time  in  seconds that you allow the connection to the server to take.  This only limits the connection phase, once curl has connected this
              option is of no more use. See also the -m/--max-time option.

```

## strace の結果

fcntl(2) O_NONBLOCK + poll(2) で待つ

```
socket(PF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
setsockopt(3, SOL_SOCKET, SO_KEEPALIVE, [1], 4) = 0
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
connect(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("192.168.100.1")}, 16) = -1 EINPROGRESS (Operation now in progress)
poll([{fd=3, events=POLLOUT|POLLWRNORM}], 1, 5000) = 0 (Timeout)
getsockopt(3, SOL_SOCKET, SO_ERROR, [-4294967296], [4]) = 0
close(3)                                = 0
```

タイムアウトした際のエラーコードは 28 で man curl にも説明が載っている

```
       28     Operation timeout. The specified time-out period was reached according to the conditions.
```

# curl --max-time

## USAGE

--max-time は実時間で curl の実行を制御する。「curlを実行してからN秒」でタイムアウトする

```
       -m/--max-time <seconds>
              Maximum  time  in  seconds  that you allow the whole operation to take.  This is useful for preventing your batch jobs from hanging for hours due to
              slow networks or links going down.  See also the --connect-timeout option.
```

デカいファイルをダウンロードしている際にタイムアウトする例

```
$ time curl --max-time 5 https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.13.6.tar.xz >/dev/null
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  7 73.6M    7 5494k    0     0   956k      0  0:01:18  0:00:05  0:01:13 1250k
curl: (28) Operation timed out after 5000 milliseconds with 5626456 out of 77194340 bytes received

real	0m5.751s
user	0m0.146s
sys	0m0.100s
```

--max-time の実体は alarm(2) である

```
alarm(5)                              = 0
```

 * コマンドの実行時間を制限するのだから select(2) とか poll(2) だと実現できなそう
 * --read-timeout や --connect-timeout と組み合わせて実装することを考えると alarm(2) 使うしか無さそう

なるほど感高いなー 

# wget --connect-timeout

## USAGE

```
       --connect-timeout=seconds
           Set the connect timeout to seconds seconds.  TCP connections that take longer to establish will be aborted.  By default, there is no connect timeout,
           other than that implemented by system libraries.
```

connect(2) のタイムアウトを指定してテスト

```
$ wget --connect-timeout 5 192.168.100.1
--2014-03-14 15:31:17--  http://192.168.100.1/
Connecting to 192.168.0.90:80... failed: Connection timed out.
Retrying.

--2014-03-14 15:31:23--  (try: 2)  http://192.168.100.1/
Connecting to 192.168.0.90:80... failed: Connection timed out.
Retrying.

--2014-03-14 15:31:30--  (try: 3)  http://192.168.100.1/
Connecting to 192.168.0.90:80... failed: Connection timed out.
Retrying.
```

wget は勝手に retry するのであった

 * wget 自身の retry
 * SYN再送

と2つのコンテキストで ___リトライ___ が行われることになる 

### strace の結果

--connect-timeout の実体は setitimer(ITTIMER_REAL) 

```
socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 3
rt_sigaction(SIGALRM, {0x4291b0, [ALRM], SA_RESTORER|SA_RESTART, 0x7f0282b6e9a0}, {SIG_DFL, [ALRM], SA_RESTORER|SA_RESTART, 0x7f0282b6e9a0}, 8) = 0
rt_sigprocmask(SIG_BLOCK, NULL, [], 8)  = 0
setitimer(ITIMER_REAL, {it_interval={0, 0}, it_value={5, 0}}, NULL) = 0
connect(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("192.168.100.1")}, 16) = ? ERESTARTSYS (To be restarted)
--- SIGALRM (Alarm clock) @ 0 (0) ---
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigaction(SIGALRM, {SIG_DFL, [ALRM], SA_RESTORER|SA_RESTART, 0x7f0282b6e9a0}, {0x4291b0, [ALRM], SA_RESTORER|SA_RESTART, 0x7f0282b6e9a0}, 8) = 0
close(3)                                = 0
```

# nc

```
nc 192.168.100.1 80
```

### strace の結果

nc はデフォルトではタイムアウト値を指定しない

```
connect(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("192.168.100.1")}, 16) = -1 EINPROGRESS (Operation now in progress)
select(4, NULL, [3], NULL, NULL)        = 1 (out [3])
getsockopt(3, SOL_SOCKET, SO_ERROR, [6543341856086818926], [4]) = 0
fcntl(3, F_SETFL, O_RDWR)               = 0
close(3)                                = 0
```

しかし、システムコール呼び出し側でタイムアウト値をしていしなくとも SYN の再送が先にタイムアウトにいたる

```
15:34:31.679754 IP 10.0.2.15.38186 > 192.168.100.1.http: Flags [S], seq 3071784278, win 14600, options [mss 1460,sackOK,TS val 84560732 ecr 0,nop,wscale 7], length 0
15:34:32.680423 IP 10.0.2.15.38186 > 192.168.100.1.http: Flags [S], seq 3071784278, win 14600, options [mss 1460,sackOK,TS val 84561733 ecr 0,nop,wscale 7], length 0
15:34:34.681910 IP 10.0.2.15.38186 > 192.168.100.1.http: Flags [S], seq 3071784278, win 14600, options [mss 1460,sackOK,TS val 84563734 ecr 0,nop,wscale 7], length 0
15:34:38.681501 IP 10.0.2.15.38186 > 192.168.100.1.http: Flags [S], seq 3071784278, win 14600, options [mss 1460,sackOK,TS val 84567734 ecr 0,nop,wscale 7], length 0
15:34:46.682595 IP 10.0.2.15.38186 > 192.168.100.1.http: Flags [S], seq 3071784278, win 14600, options [mss 1460,sackOK,TS val 84575735 ecr 0,nop,wscale 7], length 0
15:35:02.683394 IP 10.0.2.15.38186 > 192.168.100.1.http: Flags [S], seq 3071784278, win 14600, options [mss 1460,sackOK,TS val 84591735 ecr 0,nop,wscale 7], length 0
```

再送の間隔が ___1 -> 2 -> 4 -> 8 -> 16___ ( http://d.hatena.ne.jp/rx7/20131129/p1 も読んでね )

# nc -w 

```
nc -w 10 192.168.100.0 80
```

## USAGE

```
     -w timeout
             If a connection and stdin are idle for more than timeout seconds, then the connection is silently closed.  The -w flag has no effect on the -l
             option, i.e. nc will listen forever for a connection, with or without the -w flag.  The default is no timeout.
```

connection と stdin の idle 時間のタイムアウトらしい

### strace の結果

connect(2) で待つ場合は select(2) の待ち時間を指すようだ

```
socket(PF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
connect(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("192.168.100.0")}, 16) = -1 EINPROGRESS (Operation now in progress)
select(4, NULL, [3], NULL, {10, 0})     = 0 (Timeout)
```

connect(2) した後は poll(2) で待つ時間を指すようだ。不思議

```
socket(PF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
connect(3, {sa_family=AF_INET, sin_port=htons(8080), sin_addr=inet_addr("127.0.0.1")}, 16) = -1 EINPROGRESS (Operation now in progress)
select(4, NULL, [3], NULL, {10, 0})     = 1 (out [3], left {9, 999998})
getsockopt(3, SOL_SOCKET, SO_ERROR, [5936974387507888128], [4]) = 0
fcntl(3, F_SETFL, O_RDWR)               = 0
poll([{fd=3, events=POLLIN}, {fd=0, events=POLLIN}], 2, 10000) = 0 (Timeout)
```

# wget --dns-timeout

## USAGE

```
       --dns-timeout=seconds
           Set the DNS lookup timeout to seconds seconds.  DNS lookups that don’t complete within the specified time will fail.  By default, there is no timeout
           on DNS lookups, other than that implemented by system libraries.
```

--dns-timeout は libresolv のタイムアウトも絡むのでややこしい

 * DNS だけでなくて , /etc/hosts を読み込む時間、nscd にクエリを出す時間も含まれる
 * --dns-timeout > libresolv の場合は libreolsv (libnss_dns ?) のタイムアウトが優先される
  * タイムアウトの時間は /etc/resolv.conf に左右される (デフォルトでは５秒のタイムアウト timeout:<N> + リトライ attempts:<N> )
    * see also http://linuxjm.sourceforge.jp/html/LDP_man-pages/man5/resolver.5.html
  * リゾルバに左右されるので無闇に --dns-timeout を長くしても意味は無いことが分かる
 * リゾルバのソケットを select(2) や poll(2) できない??? ので setitimer(2) でタイムアウトしている
   * 個別に見るの大変そうだし ...

nameserver に適当に不達のIP を指定してタイムアウトのテストすると良い

### strace の結果

```
setitimer(ITIMER_REAL, {it_interval={0, 0}, it_value={999, 0}}, NULL) = 0

// nscd にクエリを投げようとする
socket(PF_FILE, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 3
connect(3, {sa_family=AF_FILE, path="/var/run/nscd/socket"}, 110) = -1 ENOENT (No such file or directory)
close(3)                                = 0

// DNS にクエリを投げる
socket(PF_INET, SOCK_DGRAM|SOCK_NONBLOCK, IPPROTO_IP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("192.168.100.0")}, 16) = 0
poll([{fd=3, events=POLLOUT}], 1, 0)    = 1 ([{fd=3, revents=POLLOUT}])
sendto(3, "\346\374\1\0\0\1\0\0\0\0\0\0\3www\6kernel\3org\0\0\1\0\1", 32, MSG_NOSIGNAL, NULL, 0) = 32
poll([{fd=3, events=POLLIN|POLLOUT}], 1, 5000) = 1 ([{fd=3, revents=POLLOUT}])
sendto(3, "\232\275\1\0\0\1\0\0\0\0\0\0\3www\6kernel\3org\0\0\34\0\1", 32, MSG_NOSIGNAL, NULL, 0) = 32
poll([{fd=3, events=POLLIN}], 1, 4999)  = ? ERESTART_RESTARTBLOCK (To be restarted)
--- SIGALRM (Alarm clock) @ 0 (0) ---
```

# wget --read-timeout

## USAGE

```
       --read-timeout=seconds
           Set the read (and write) timeout to seconds seconds.  The "time" of this timeout refers to idle time: if, at any point in the download, no data is
           received for more than the specified number of seconds, reading fails and the download is restarted.  This option does not directly affect the duration
           of the entire download.
```

idle時間のタイムアウトを指定する。
 * idle = データを受信しない時間
 * connect(2) した後のデータ読む段階での select(2) でタイムアウトの時間を指す

nc -l 8080 で accept(2) だけする疑似 httpd に向けて wget してテストする。下記はその結果

```
$ wget --read-timeout=10 127.0.0.1:8080
--2014-03-14 23:20:08--  http://127.0.0.1:8080/
Connecting to 127.0.0.1:8080... connected.
HTTP request sent, awaiting response... Read error (Connection timed out) in headers.
Retrying.

--2014-03-14 23:20:19--  (try: 2)  http://127.0.0.1:8080/
Connecting to 127.0.0.1:8080... failed: Connection refused.
```

### strace の結果

```
socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(8080), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
write(2, "connected.\n", 11connected.
)            = 11
select(4, NULL, [3], NULL, {10, 0})     = 1 (out [3], left {9, 999998})
write(3, "GET / HTTP/1.0\r\nUser-Agent: Wget"..., 112) = 112
write(2, "HTTP request sent, awaiting resp"..., 40HTTP request sent, awaiting response... ) = 40
select(4, [3], NULL, NULL, {10, 0})     = 0 (Timeout)
```

# wget --timeout=seconds

## USAGE

```
       --timeout=seconds
           Set the network timeout to seconds seconds.  This is equivalent to specifying --dns-timeout, --connect-timeout, and --read-timeout, all at the same
           time.
```

 * 3つまとめてセットくん
 * 大雑把過ぎる気がするので、個別に指定した方がトラブル時の調査も楽なのかな?

# mysql

```
[vagrant@vagrant-centos65 ~]$ mysql --version
mysql  Ver 14.14 Distrib 5.1.73, for redhat-linux-gnu (x86_64) using readline 5.1
```

## mysql オプション無し

connect(2) でタイムアウト無し。TCP再送の上限までブロックする

```
$ mysql -h 192.168.100.1
ERROR 2003 (HY000): Can't connect to MySQL server on '192.168.100.1' (110)

# (110) = ETIMEDOUT
#     #define ETIMEDOUT       110     /* Connection timed out */
#
```

## strace の結果

```
socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 3
fcntl(3, F_SETFL, O_RDONLY)             = 0
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
connect(3, {sa_family=AF_INET, sin_port=htons(3306), sin_addr=inet_addr("192.168.100.1")}, 16) = -1 ETIMEDOUT (Connection timed out)
shutdown(3, 2 /* send and receive */)   = -1 ENOTCONN (Transport endpoint is not connected)
close(3)                                = 0
```

## mysql ---connect_timeout=5

```
$ mysql -h 192.168.100.1 --connect_timeout=5
ERROR 2003 (HY000): Can't connect to MySQL server on '192.168.100.1' (4)
```

### strace の結果

fcntl(2) O_NONBLOCK + poll(2) を使ってタイムアウトを実装している

```
socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 3
fcntl(3, F_SETFL, O_RDONLY)             = 0
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
connect(3, {sa_family=AF_INET, sin_port=htons(3306), sin_addr=inet_addr("192.168.100.1")}, 16) = -1 EINPROGRESS (Operation now in progress)
fcntl(3, F_SETFL, O_RDWR)               = 0
poll([{fd=3, events=POLLIN|POLLPRI}], 1, 5000) = 0 (Timeout)
shutdown(3, 2 /* send and receive */)   = 0
close(3)                                = 0
```

## connect(2) 後のタイムアウト

nc -l 3306 で accept(2) だけするサーバーをたててテスト。半永久的にブロックする

```
$ mysql -h127.0.0.1
```

### strace の結果

connect(2) 後の read(2) でブロックする。認証か何かするための read(2) かな?

```
socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 3
fcntl(3, F_SETFL, O_RDONLY)             = 0
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
connect(3, {sa_family=AF_INET, sin_port=htons(3306), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
setsockopt(3, SOL_SOCKET, SO_RCVTIMEO, "\2003\341\1\0\0\0\0\0\0\0\0\0\0\0\0", 16) = 0
setsockopt(3, SOL_SOCKET, SO_SNDTIMEO, "\2003\341\1\0\0\0\0\0\0\0\0\0\0\0\0", 16) = 0
setsockopt(3, SOL_IP, IP_TOS, [8], 4)   = 0
setsockopt(3, SOL_TCP, TCP_NODELAY, [1], 4) = 0
setsockopt(3, SOL_SOCKET, SO_KEEPALIVE, [1], 4) = 0
read(3, 
```

SO_RCVTIMEO, SO_SNDTIMEO を指定しているけど、作用してない?

# mysql --connect_timeout

nc -l 3306 で accept(2) だけするサーバーをたててテスト

```
$ mysql -h 127.0.0.1 --connect_timeout=30

ERROR 2003 (HY000): Can't connect to MySQL server on '127.0.0.1' (4)

# (4) = EINTR ? 
#     #define EINTR            4      /* Interrupted system call */
# 
```

connect(2) が成功したかどうかは確認せず poll(2) に入る 
 * ノンブロッキングなので connect(2) の結果は見てないけど、ソケットは ESTABLISED になってる
 * connect_timeout を指定しないと read(2) で永遠にブロックしてたけど、こちらはタイムアウトする

```
socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 3
fcntl(3, F_SETFL, O_RDONLY)             = 0
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
connect(3, {sa_family=AF_INET, sin_port=htons(3306), sin_addr=inet_addr("127.0.0.1")}, 16) = -1 EINPROGRESS (Operation now in progress)
fcntl(3, F_SETFL, O_RDWR)               = 0
poll([{fd=3, events=POLLIN|POLLPRI}], 1, 30000
```

## mysql --connect_timeout の有無でエラーコードが違う

3306 で listen(2) しているプロセスはいない状態で比較
 * ECONNREFUSED を返されている
```
[vagrant@vagrant-centos65 ~]$ mysql -h 127.0.0.1
ERROR 2003 (HY000): Can't connect to MySQL server on '127.0.0.1' (111)

# (111) は errno を指す
#    #define ECONNREFUSED    111     /* Connection refused */
#
```

connect_timeout を指定してるとなかなか見慣れないメッセージになる。戸惑いそう
  * errno が 111 = ECONNREFUSED なのを把握できればよさそ
```
[vagrant@vagrant-centos65 ~]$ mysql -h 127.0.0.1 --connect_timeout=1
ERROR 2013 (HY000): Lost connection to MySQL server at 'reading initial communication packet', system error: 111
```

### サーバ無し、mysql --connect_timeout の strace の結果

O_NONBLOCK なので errno がややこしい

 * connect(2)  EINPROGRESS
 * read(2)     ECONNREFUSED
 * shutdown(2) ENOTCONN

を返している 

```
socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 3
fcntl(3, F_SETFL, O_RDONLY)             = 0
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
connect(3, {sa_family=AF_INET, sin_port=htons(3306), sin_addr=inet_addr("127.0.0.1")}, 16) = -1 EINPROGRESS (Operation now in progress)
fcntl(3, F_SETFL, O_RDWR)               = 0
poll([{fd=3, events=POLLIN|POLLPRI}], 1, 1000) = 1 ([{fd=3, revents=POLLIN|POLLERR|POLLHUP}])
setsockopt(3, SOL_SOCKET, SO_RCVTIMEO, "\2003\341\1\0\0\0\0\0\0\0\0\0\0\0\0", 16) = 0
setsockopt(3, SOL_SOCKET, SO_SNDTIMEO, "\2003\341\1\0\0\0\0\0\0\0\0\0\0\0\0", 16) = 0
setsockopt(3, SOL_IP, IP_TOS, [8], 4)   = 0
setsockopt(3, SOL_TCP, TCP_NODELAY, [1], 4) = 0
setsockopt(3, SOL_SOCKET, SO_KEEPALIVE, [1], 4) = 0
poll([{fd=3, events=POLLIN}], 1, 1000)  = 1 ([{fd=3, revents=POLLIN|POLLERR|POLLHUP}])
read(3, 0x118eb90, 16384)               = -1 ECONNREFUSED (Connection refused)
shutdown(3, 2 /* send and receive */)   = -1 ENOTCONN (Transport endpoint is not connected)
close(3)                                = 0
```

# より道

```
$ mysql
ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock' (2)

# (2) は ENOENT
#    define ENOENT           2      /* No such file or directory */
# UNIXドメインソケットの有無
```

# tips

loopback の tcpdump では `-i lo` 忘れずに

```
sudo tcpdump -i lo port 3306
```
 
