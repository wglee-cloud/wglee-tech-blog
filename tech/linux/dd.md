---
description: 2022. 8. 9. 01:34
---

# dd 명령어 개념과 손쉽게 더미 파일 생성하기

## Intro

dd 명령어는 파일을 복사하고 변환할 수 있는 명령어이다.\
Linux에서는 하드 디스크도 파일로 간주 된다. dd 는 이러한 하드 디스크 및 파티션을 복제할 수 있다는 데서 사용성이 높다. \
하드 디스크를 제어할 수 있는 만큼 주의 깊게 사용 해야 한다.

dd 명령어를 이용하여 다음을 수행할 수 있다.

* 디스크 복제
* 파티션 복제
* 전체 하드 디스크 또는 파티션 백업 및 복원
* 하드 디스크 데이터 삭제



## Syntax

dd 명령어의 기본적인 문법과 사용법은 다음과 같다.

```shell-session
root@DESKTOP:~# dd --help
Usage: dd [OPERAND]...
  or:  dd OPTION
Copy a file, converting and formatting according to the operands.

  bs=BYTES        read and write up to BYTES bytes at a time (default: 512);
                  overrides ibs and obs
  cbs=BYTES       convert BYTES bytes at a time
  conv=CONVS      convert the file as per the comma separated symbol list
  count=N         copy only N input blocks
  ibs=BYTES       read up to BYTES bytes at a time (default: 512)
  if=FILE         read from FILE instead of stdin
  iflag=FLAGS     read as per the comma separated symbol list
  obs=BYTES       write BYTES bytes at a time (default: 512)
  of=FILE         write to FILE instead of stdout
  oflag=FLAGS     write as per the comma separated symbol list
  seek=N          skip N obs-sized blocks at start of output
  skip=N          skip N ibs-sized blocks at start of input
  status=LEVEL    The LEVEL of information to print to stderr;
                  'none' suppresses everything but error messages,
                  'noxfer' suppresses the final transfer statistics,
                  'progress' shows periodic transfer statistics

N and BYTES may be followed by the following multiplicative suffixes:
c =1, w =2, b =512, kB =1000, K =1024, MB =1000*1000, M =1024*1024, xM =M,
GB =1000*1000*1000, G =1024*1024*1024, and so on for T, P, E, Z, Y.
```

```shell-session
# dd if=source-disk of=destination-disk [option]
```

`if` : input file의 약자. source 파일의 위치를 지정한다. 디스크 복제의 경우, /dev/vdc 와 같이 지정할 수 있다.\
`of` : output file의 약자. 복제한 파일들을 저장할 위치를 지정한다.\
`bs` : block size로, 한번에 읽어들일 bytes 수를 의미한다. 기본값은 512bytes이다. block size가 클 수록 output file에 데이터가 더 빠르게 쓰인다. ex. bs=2048 (bytes 단위), bs=2K (kilobytes 단위)\
`count` : 복제할 블록의 수를 의미한다. bs \* count 값을 하면 복사하는 데이터의 전체 크기가 된다.\
`status` : dd 작업의 진행 상황을 볼 수 있다. progress로 지정하면 복사 중인 block 을 표준 출력으로 보여준다.



🛑 block size가 클수록 속도가 빨라진다고 해서 무조건 수치를 높게 해서는 안된다. write 하는 장치가 무엇인지에 따라 적정한 값을 설정해야 한다. ( ex. cd, dvd 인지 hard disk 인지 등등 )



## dd로 더미 파일 생성하기

/dev 디렉터리 하위에 있는 /dev/zero나 /dev/urandom 같은 special directory를 사용하여 더미 파일을 생성할 수 있다.\
테스트할 때 원하는 크기의 파일을 쉽게 생성하는데 유용하다.

```shell-session
root@DESKTOP:~/script# dd if=/dev/zero of=dummy.txt bs=512 count=10
10+0 records in
10+0 records out
5120 bytes (5.1 kB, 5.0 KiB) copied, 0.0103007 s, 497 kB/s

root@DESKTOP:~/script# ls -alh
total 20K
drwxr-xr-x 1 wglee wglee 4.0K Aug  9 01:21 .
drwxr-xr-x 1 wglee wglee 4.0K Aug  9 00:13 ..
-rw-r--r-- 1 wglee wglee 5.0K Aug  9 01:21 dummy.txt
-rw-r--r-- 1 wglee wglee   57 Mar 22 23:39 test.txt
```



## **참고 문서**

{% embed url="https://www.maketecheasier.com/use-dd-command-linux/" %}

{% embed url="http://www.makelinux.net/alp/046.htm" %}

{% embed url="https://www.baeldung.com/linux/dd-command" %}

{% embed url="https://www.geeksforgeeks.org/dd-command-linux/" %}
