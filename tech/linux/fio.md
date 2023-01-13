---
description: 2022. 5. 23. 21:02
---

# fio 이용하여 디스크 성능 측정하기

## Intro

오늘 fio 라는 명령어를 처음 사용해 봤다.\
fio(Flexible IO Tester)는 디스크에 I/O 부하를 줘서 성능을 측정할 수 있는 도구이다.

disk 성능 테스트의 목적은 물리 디바이스가 잘 동작하는지 1차적으로 확인하는 것이다.\
그리고 application을 올린 다음에 다시 테스트 하여 본래의 성능 보다 너무 낮으면 application 설정의 튜닝을 고려하는 지표로도 사용할 수 있다고 한다.

fio를 사용하면 random read/write 방식으로 disk에 부하를 줄 수 있다.



## ramdom read/write

disk read/write 방식에는 순차(sequential)/랜덤(random) 방식이 있다.\
random read/write가 일어나는 상황은 다음과 같이 가정할 수 있다. 여러 프로세스가 동시에 동작하며 disk를 사용할 때 하나의 서비스에 대한 파일들이 디스크의 여기저기에 흩어져서 저장되게 된다. 보통 PC나 서버를 사용할 때 한번에 하나의 작업만 하는 경우는 거의 없다. (ex. PC 카카오톡 하면서 크롬으로 유튜브 보기) 따라서 random read/write 방식이 실제 환경의 접근 패턴에 더 가깝다고 할 수 있다. fio는 random read/write를 지원한다.



## 옵션 설명

다음 명령어는 /dev/sdb 디스크의 10GB에 block 4KiB 단위로 데이터를 60분간 read 한다는 뜻이다.\
세부적인 옵션도 한번 알아보도록 한다.

```shell-session
# fio  --filename=/dev/sdb --direct=1 --sync=1 --rw=randread  --stonewall  --bs=4k --numjobs=16 --iodepth=16 --runtime=60 --time_based --group_reporting --name=bigt --size=10G
```

* \--filename : 부하를 줄 위치를 지정한다
* \--direct : 1(true)이면 non-buffered I/O이다. default는 0(false)
* \--sync : 1(true)이면 write 작업에 대해 동기화를 한다. 데이터가 캐시 같이 휘발성의 공간이 아닌 디스크에 쓰여졌을 때 call 을 반환한다.
* \--rw : I/O 패턴을 지정한다. read/write/randread/randwrite 등의 옵션이 있다.
* \--stonewall : 진행 중인 job이 완료될 때까지 새로운 job을 시작 안하고 대기한다.
* \--bs : 제일 중요한 것 같다. Block Size를 의미한다. read 혹은 write할 block의 크기를 의미한다. 기본값은 4KiB(=4096byte)이다.
* \--numjobs : 주어진 작업을 동시에 진행할 독립적인 thread나 process의 개수를 의미한다.
* \--runtime : 부하를 줄 시간(초단위)
* \--time\_based : runtime으로 지정한 시간 안에 지정한 부하가 끝나더라도 설정한 시간이 될 때까지 테스트를 반복한다.
* \--group\_reporting : 결과를 job이 아닌 group 별로 보여준다.
* \--name : 해당 job의 이름을 지정한다.
* \--size : 해당 job에서 부하 줄 디스크 용량을 지정한다. 만약에 지정을 안하면 명시한 디바이스 전체에 대해 부하 테스트를 한다.

아래 명령어도 비슷하다. 다만 randwrite로 disk write 작업을 한다.&#x20;

```shell-session
# fio --filename=/dev/sdb  --direct=1 --sync=1 --rw=randwrite  --stonewall  --bs=4k --numjobs=16 --iodepth=16 --runtime=60 --time_based --group_reporting --name=bigt --size=10G
```



## 결과

이렇게 부하를 주면 결과는 다음과 같이 나타난다.

```shell-session
[root@wglee-testserver ~]# fio  --filename=/dev/vdb --direct=1 --sync=1 --rw=randread  --stonewall  --bs=4k --numjobs=16 --iodepth=16 --runtime=60 --time_based --group_reporting --name=bigt --size=10G
bigt: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=16
fio-3.7
Starting 16 processes
Jobs: 16 (f=16): [r(16)][100.0%][r=59.0MiB/s,w=0KiB/s][r=15.1k,w=0 IOPS][eta 00m:00s]
bigt: (groupid=0, jobs=16): err= 0: pid=15066: Mon May 23 16:01:25 2022
   read: IOPS=12.7k, BW=49.5MiB/s (51.9MB/s)(2971MiB/60014msec)
    clat (usec): min=53, max=681160, avg=1257.59, stdev=6374.48
     lat (usec): min=54, max=681161, avg=1258.25, stdev=6374.50
    clat percentiles (usec):
     |  1.00th=[   355],  5.00th=[   490], 10.00th=[   570], 20.00th=[   676],
     | 30.00th=[   766], 40.00th=[   848], 50.00th=[   938], 60.00th=[  1037],
     | 70.00th=[  1156], 80.00th=[  1319], 90.00th=[  1614], 95.00th=[  2008],
     | 99.00th=[  4490], 99.50th=[  6718], 99.90th=[ 49546], 99.95th=[104334],
     | 99.99th=[316670]
   bw (  KiB/s): min=   16, max= 4960, per=6.26%, avg=3171.61, stdev=917.69, samples=1918
   iops        : min=    4, max= 1240, avg=792.88, stdev=229.42, samples=1918
  lat (usec)   : 100=0.06%, 250=0.46%, 500=5.02%, 750=22.65%, 1000=28.47%
  lat (msec)   : 2=38.29%, 4=3.84%, 10=0.93%, 20=0.10%, 50=0.07%
  lat (msec)   : 100=0.05%, 250=0.04%, 500=0.01%, 750=0.01%
  cpu          : usr=0.45%, sys=1.90%, ctx=760556, majf=0, minf=622
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=760583,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=16

Run status group 0 (all jobs):
   READ: bw=49.5MiB/s (51.9MB/s), 49.5MiB/s-49.5MiB/s (51.9MB/s-51.9MB/s), io=2971MiB (3115MB), run=60014-60014msec

Disk stats (read/write):
  vdb: ios=758922/0, merge=0/0, ticks=893372/0, in_queue=889486, util=99.40%
```

결과 데이터가 엄청 많은데 주로 이 부분을 보면 될 것 같다.\
초당 몇 MB의 데이터를 read 했는지 알 수 있다.

```shell-session
read: IOPS=12.7k, BW=49.5MiB/s (51.9MB/s)
```

동일 서버에 세션을 하나 더 열어서 sar로 fio 명령으로 인해 디스크에 write 혹은 read 되는 값을 확인 할 수도 있다.

```shell-session
[root@wglee-testserver sa]# sar -dp 1 | grep vdb
04:00:30 PM       DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
04:00:34 PM       vdb  12856.94 102855.56      0.00      8.00     20.89      1.60      0.11    138.61
04:00:35 PM       vdb  13887.67 111101.37      0.00      8.00     21.09      1.48      0.10    136.58
04:00:36 PM       vdb  18297.06 146376.47      0.00      8.00     22.49      1.29      0.08    146.76
04:00:37 PM       vdb  15250.70 122005.63      0.00      8.00     21.26      1.37      0.09    140.42
```

##

## 참고 문서

{% embed url="https://fio.readthedocs.io/en/latest/fio_doc.html" %}

{% embed url="https://www.systutorials.com/docs/linux/man/1-fio/" %}



