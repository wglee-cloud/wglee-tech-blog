---
description: 2022. 8. 11. 01:52
---

# Shell Script 짜면서 찾아봤던 것들

## Intro

쉘 스크립트를 작성하면서 매번 찾아봐야 했거나, 제대로 알고 넘어가야겠다고 생각했던 구문들에 대해 정리해본다.\
파편적인 내용들이라서 도움이 많이 안 될 수도 있으나 이미 경험한 것이라도 까먹지 말아야겠다는 생각에 기록한다.

<figure><img src="https://blog.kakaocdn.net/dn/NFCH5/btrJqmr1YT7/2DVOrDQwFuhYcJ6zCT2WK0/img.png" alt=""><figcaption></figcaption></figure>



## loop 돌면서 파일의 line 읽기

```shell
while read line;
do
  echo '$line'
done < /home/wglee/list.txt
```

이런 식으로도 할 수 있다.\
더 나아가서 주석 구문과 같이 cat한 내용을 표준 입력으로 넘겨 sort, uniq 를 할 수도 있다.

<pre class="language-shell"><code class="lang-shell"><strong># cat /home/wglee/list.txt | sort | uniq | while read line
</strong>cat /home/wglee/list.txt | while read line
do
  echo '$line'
done
</code></pre>



## 문자열에 특정 단어가 포함되어 있는지 확인하고 싶을 때

이중 대괄호를 이용하면 와일드카드( \* ) 문자열 패턴 매칭을 확인할 수 있다.\
단일 대괄호에 비해 향상된 기능이다.

```shell
if [[ "There is a cat" == *"cat"* ]]
then
  echo "String containes a word 'cat'"
fi
```



## if 조건식에서 대괄호 한개는 뭐고 대괄호 두개는 뭐지?

읽어보자..\
단일 대괄호는 if test 구문과 같다.\
[https://unix.stackexchange.com/questions/306111/what-is-the-difference-between-the-bash-operators-vs-vs-vs](https://unix.stackexchange.com/questions/306111/what-is-the-difference-between-the-bash-operators-vs-vs-vs)&#x20;



## eval 명령어 사용해서 shell script 에서 openstack 명령어 실행하기

eval 명령어는 자신의 인자를 shell 명령어로 실행시킨다.\
커맨드를 변수에 문자열로 저장하고, 그 변수를 shell 상에 실행시키고자 할 때 유용하다.\
예를 들어서 나는 이런식으로 사용했다.\
filter 라는 변수에 담긴 오픈스택 명령어를 eval 의 인자로 줘서 실행시키고, 그 결과값을 파이프 (|) 로 넘겨서 awk 으로 원하는 값을 뽑아내었다.

```shell
#해당 스냅샷 이름으로 검색 (같은 이름의 스냅샷의 경우 여러 항목 출력)
filter="openstack volume snapshot list --all-project | grep -w $name"


#스냅샷 아이디와 스냅샷 이름 awk로 필터링
#awk에서 '{ printf("%s\t%s\n", $2, $3) }' 이렇게 하면 두 값 사이에 tab을 출력한다.
snapshot_id="$(eval $filter | awk -F '|' '{ printf("%s\t%s\n", $2, $3) }')"
```



## escape 문자 활용해서 echo 예쁘게 찍기

echo에 -e 옵션을 주지 않으면 escape 문자를 그냥 문자열로 출력한다.

```shell-session
root@deploy:~# cat wglee.sh
#!/bin/bash
echo -e "Hello\nNice\tto\tmeet\tyou!"  

root@deploy:~# . wglee.sh  
Hello  
Nice     to     meet     you!
```



## sed 이용하여 파일 중간에 문자열 삽입하기

sed를 사용하여 문자열 치환 뿐만 아니라 파일의 원하는 위치에 insert를 할 수도 있다. sed에 -i 옵션을 주면 변경 사항을 파일에 저장한다는 뜻이다. '**1**i\문자열' 이면 첫번째 행에 문자열을 추가한다. 이런식으로 insert할 행의 숫자를 지정할 수 있다.

```shell-session
root@deploy:# cat -n list.txt  
1 Do you  
2 know how to  
3 insert a string using sed?  

root@deploy:~# sed -i '1i\\Hello ~' list.txt  
root@deploy:~\# sed -i '3i\\Hi~' list.txt  
root@deploy:~\# cat -n list.txt  
1 Hello~  
2 Do you  
3 Hi ~  
4 know how to  
5 insert a string using sed?
```



## 파일을 escape 문자 기준으로 확장하여 예쁘게 저장하기

pr 명령어를 이번에 처음 알았다. man 페이지에서는 다음과 같이 설명한다.\
"pr - convert text files for printing"\
정확히는 모르겠지만 프린트하기 전에 문자열을 정비하기 좋은 명령어인 듯 하다.\
\-e 옵션은 지정한 숫자로 tab을 확장한다.

```shell-session
root@deploy:~# cat list.txt  
This\tis\ta\tsentence

root@deploy:~# echo -e `cat list.txt`  
This   is   a   sentence

root@deploy:~# echo -e `cat list.txt` | pr -t -e5  
This is a sentence

root@deploy:~# echo -e `cat list.txt` | pr -t -e10  
This         is       a      sentence
```



## 문자열을 array로 변환하고 특정 index 에 접근하기

아래와 같이 공백을 기준으로 문자열을 배열로 바꿀 수 있다. 배열의 길이나 특정 index의 값도 구할 수 있다.

```shell-session
root@deploy:~# cat wglee.sh
#!/bin/bash

httpd_pid="1234 5678 91011 121314 151617 181920"
echo -e "$httpd_pid"

read -a httpd_pid_array <<< $httpd_pid

httpd_pid_count=${#httpd_pid_array[@]}
echo "count : $httpd_pid_count"
echo "print all : ${httpd_pid_array[@]}"
echo "print value which index is 1 : ${httpd_pid_array[1]}"

root@deploy:~# . wglee.sh
1234 5678 91011 121314 151617 181920
count : 6
print all : 1234 5678 91011 121314 151617 181920
print value which index is 1 : 5678
```



## 참고 문서

나의 수많은 삽질과 stackoverflow 사이트들

