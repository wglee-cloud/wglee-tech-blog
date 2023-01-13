# fallocate 명령어로 만든 파일의 mkswap 불가 현상 (sparse file)

## Intro

fallocate 명령어로 생성한 파일을 swap 영역으로 사용하려 했는데 swapon을 할 때 다음과 같은 에러가 발생했다.

> **swapon: /root/swapfile: swapon failed: Invalid argument**

```shell-session
[root@server-1-lab ~]# fallocate -l 200MiB /root/swapfile

[root@server-1-lab ~]# ls -lh | grep swap
-rw-r--r--.  1 root root 200M Aug 28 19:04 swapfile

[root@server-1-lab ~]# chmod 600 swapfile

[root@server-1-lab ~]# mkswap /root/swapfile
Setting up swapspace version 1, size = 204796 KiB
no label, UUID=f7c9b365-c7d0-400e-a403-2894447da28e

[root@server-1-lab ~]# free -h
              total        used        free      shared  buff/cache   available
Mem:           990M        152M        439M         56M        398M        634M
Swap:          119M          0B        119M

[root@server-1-lab ~]# swapon /root/swapfile
swapon: /root/swapfile: swapon failed: Invalid argument
```



이때 dmesg 를 보면 커널에서 다음과 같은 로그가 발생했다.&#x20;

```shell-session
[root@server-1-lab ~]# dmesg | tail
[98860.730680] Adding 204796k swap on /root/swapfile.  Priority:-3 extents:2 across:468588k FS
...
[104589.765986] swapon: swapfile has holes
```

swapon 의 man 페이지를 보면 다음과 같은 문구가 있다.

{% hint style="info" %}
**Files with holes**\
****The swap file implementation in the kernel expects to be able to write to the file directly, without the assistance of the filesystem. This is a problem on files with holes or on copy- on-write files on filesystems like Btrfs. Commands like cp(1) or truncate(1) create files with holes. These files will be rejected by swapon. **Preallocated files created by fallocate(1) may be interpreted as files with holes too depending of the filesystem.** Preallocated swap files are supported on XFS since Linux 4.18. **The most portable solution to create a swap file is to use dd(1) and /dev/zero**.
{% endhint %}



fallocate으로 생성한 Preallocated 파일은 파일시스템에 따라 "files with holes"로 인식되어 swap 으로 사용할 수 없을 수 있다고 한다.

개인적으로 해당 문제를 해결하기 위해서는\
처음부터 swapfile을 dd if=/dev/zero 로 생성하는 것이 제일 마음 편하고 깔끔하다고 생각된다.\
fallocate로 만든 파일을 `cp --sparse=never` 등으로 copy 해서 none-sparse 파일로 만드는 방법도 있다.

이제 해당 문제를 보면서 접한 `Filese with holes` 가 무슨 뜻인지 좀 더 알아보도록 하자.



<mark style="color:red;">아래 내용은 개인 공부용으로 작성한 내용입니다.</mark>\ <mark style="color:red;">내용이 틀리거나 부족할 수 있어 참고시 주의 부탁드립니다.</mark>

## Files with Holes

Files with Holes는 **Sparse File** 이라고 불리기도 한다.\
Sparse File이란, 파일 생성할 때 지정하는 크기대로 물리적인 영역을 할당하지 않고, 실질적인 데이터 write가 있을 때만 block을 할당하는 방식의 파일을 의미한다. 디스크 공간을 효율적으로 쓸 수 있다는 장점이 있다.

<figure><img src="https://blog.kakaocdn.net/dn/dqd6jA/btrKTOneIog/ArRm5jSkHIdJfq3Fj6CBk1/tfile.svg" alt=""><figcaption><p>image from&#x26;nbsp;https://en.wikipedia.org/wiki/Sparse_file</p></figcaption></figure>

위 사진과 같이 실질적으로 물리 disk를 점유한 파일 크기가 Logical 파일 크기보다 작다.





## Sparse File 생성하고 Physical / Logical Size 비교해 보기

Sparse File을 생성하는 방법은 여러가지가 있으나 나는 익숙한 dd 명령어를 사용했다.\
dd 명령어에서 seek 옵션을 주면 시작 지점에서 지정한 만큼의 block 을 건너뛴다. (hole을 만든다.)

```shell-session
[root@server-1-lab ~]# dd if=/dev/zero of=sparse_file bs=1 count=0 seek=200M
0+0 records in
0+0 records out
0 bytes (0 B) copied, 1.9797e-05 s, 0.0 kB/s
```



이렇게 생성한 파일을 ls -lsh 로 확인하면 다음과 같다.\
\--> `-s` : 물리적으로 실제로 할당된 block을 나타냄\
첫번째 항목을 보면 물리적인 파일 크기가 0 이다.

```shell-session
[root@server-1-lab ~]# ls -lskh sparse_file
0 -rw-r--r--. 1 root root 200M Aug 29 21:57 sparse_file
```



stat 명령어로도 Blocks 항목이 0 이다.\
실제로 데이터 write하여 점유한 디스크 영역이 없다.

```shell-session
[root@server-1-lab ~]# stat sparse_file
  File: 'sparse_file'
  Size: 209715200       Blocks: 0          IO Block: 4096   regular file
Device: fc01h/64513d    Inode: 4208089     Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Context: unconfined_u:object_r:admin_home_t:s0
Access: 2022-08-29 22:14:59.206901739 +0900
Modify: 2022-08-29 21:57:17.097315605 +0900
Change: 2022-08-29 21:57:17.097315605 +0900
 Birth: -
```



이제 파일에 데이터를 write 해 본다.\
다음과 같이 간단한 스크립트를 작성하여 실행 했다.

```shell-session
[root@server-1-lab ~]# cat wglee.sh
#!/bin/bash

for i in {1..500};
do
  echo "This is a Text $i" >> /root/sparse_file
done
```



Blocks 수치가 0 에서 128로 증가했다.\
다만  Logical Size도 같이 증가했는데... Logical Size는 동일해야 하지 않나 싶다. 이것은 좀 의문이다.

```shell-session
 [root@server-1-lab ~]# stat sparse_file
  File: 'sparse_file'
  Size: 209724592       Blocks: 128        IO Block: 4096   regular file
Device: fc01h/64513d    Inode: 4208089     Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Context: unconfined_u:object_r:admin_home_t:s0
Access: 2022-08-29 22:14:59.206901739 +0900
Modify: 2022-08-29 22:19:20.010679119 +0900
Change: 2022-08-29 22:19:20.010679119 +0900
 Birth: -
```



**( 추가 )**\
****hexdump 라는걸 이용해서 파일의 내용을 16진수로 확인해 보았다.\
sparse\_file을 head로 봤을 때는 zero (hole) 로 채워져 있고\
tail 로 봤을 때 문자열이 들어가 있다.\
나는 sparse 파일에 데이터 쓰기가 일어나면 zero로 채워진 hole 영역이 데이터로 치환 되는 줄 알았는데\
그게 아니라 파일의 뒷부분에 write 되는 것으로 보여진다.

```shell-session
[root@server-1-lab ~]# hexdump -vC sparse_file > wglee_sparse_hexdump
0c7fff40  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0c7fff50  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0c7fff60  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0c7fff70  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0c7fff80  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0c7fff90  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0c7fffa0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0c7fffb0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0c7fffc0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0c7fffd0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0c7fffe0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0c7ffff0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0c800000  54 68 69 73 20 69 73 20  61 20 54 65 78 74 20 31  |This is a Text 1|
0c800010  0a 54 68 69 73 20 69 73  20 61 20 54 65 78 74 20  |.This is a Text |
0c800020  32 0a 54 68 69 73 20 69  73 20 61 20 54 65 78 74  |2.This is a Text|
0c800030  20 33 0a 54 68 69 73 20  69 73 20 61 20 54 65 78  | 3.This is a Tex|
0c800040  74 20 34 0a 54 68 69 73  20 69 73 20 61 20 54 65  |t 4.This is a Te|
0c800050  78 74 20 35 0a 54 68 69  73 20 69 73 20 61 20 54  |xt 5.This is a T|
0c800060  65 78 74 20 36 0a 54 68  69 73 20 69 73 20 61 20  |ext 6.This is a |
0c800070  54 65 78 74 20 37 0a 54  68 69 73 20 69 73 20 61  |Text 7.This is a|
0c800080  20 54 65 78 74 20 38 0a  54 68 69 73 20 69 73 20  | Text 8.This is |
```



## 참고 문서

{% embed url="https://super-unix.com/unixlinux/linux-what-are-the-holes-in-files-created-with-fallocate/" %}

{% embed url="https://askubuntu.com/questions/1017309/fallocate-vs-dd-for-swapfile" %}

{% embed url="https://askubuntu.com/questions/269480/why-does-ls-l-output-a-different-size-from-ls-s" %}

{% embed url="https://meetup.toast.com/posts/37" %}
