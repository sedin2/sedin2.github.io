---
title: Redis data types
date: 2024-03-09 00:00:00 +09:00
categories: [redis, cache, 캐시]
tags: [redis]
image: /assets/img/redis.png
---

### 📌소개

**`Redis`**의 **`data type`**에 대해 알아보고 **`docker`**로 **`redis-cli`**을 통해 학습해보자.  
먼저 redis 공식 사이트([Redis data type](https://redis.io/docs/data-types/))에 가보면 다양한 redis에서 지원하는 data type이 있다.  
대표적으로 **`Strings, JSON, Lists, Sets, Hashes, Sorted Sets, Streams`** 등이 있다.

---

### 🚀Redis-cli 접속

- 먼저 Docker는 설치 되어있다고 가정하고 진행합니다.

```bash
$ docker -v                                  # 버전 확인을 통해 docker가 잘 설치된지 확인
Docker version 20.10.7, build f0df350

$ docker pull redis                          # redis 뒤에 버전이 없다면 최신버전을 pull 받아짐

$ docker run -it --name redis001 -d redis    # redis001이라는 이름으로 컨테이너를 지정하고 백그라운드(데몬)로 redis 실행

$ docker ps -a    # 해당 명령어로 잘 실행되었는지 프로세스 목록을 확인
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS      NAMES
361ba15e39bc   redis     "docker-entrypoint.s…"   3 seconds ago   Up 2 seconds   6379/tcp   redis001

$ docker exec -it redis001 bash     # redis001 컨테이너 내부로 접속!
root@361ba15e39bc:/data#            # 컨테이너에 접속하면 프롬프트에 redis001 컨테이너 ID 해쉬값 보임

root@361ba15e39bc:/data# redis-cli
127.0.0.1:6379>                     # 접속하면 IP:port> 프롬프트로 변경

127.0.0.1:6379> ping                # ping 명령어로 pong 응답 확인
PONG
```

---

### ✅Strings

- 기본 명령어  
  **`SET`** - **key-value 저장**  
  **`SETNX`** - **key가 없을때**만 저장(**Lock** 구현시 유용)  
  **`GET`** - key에 해당하는 **value값 조회**  
  **`MSET`** - **단일 명령**으로 **여러 key-value값 저장**  
  **`MGET`** - **단일 명령**으로 **여러 value값 조회**

- String 특징  
  Value 값은 **모든 종류**의 **String(바이너리 데이터 포함)** 및 jpeg와 같은 **이미지**도 가능함.  
   단, value값은 **512MB** 제한

- redis-cli 예시

```bash
> SET name:1 mocha   # key(name:1) value(mocha) 저장
OK
> GET name:1         # key(name:1)에 해당하는 value(mocha) 조회
"mocha"

> SET name:1 sejin nx   # key(name:1)가 이미 존재해서 저장 실패
(nil)                   # 만약 동시에 여러 클라이언트의 요청을 처리해야 할시,
                        # SETNX를 통해 단 1건만 저장하고 나머진 다 실패처리 가능

> SET name:2 sejin nx   # 만약 key가 없다면 해당 key로 value를 저장한 뒤, OK를 리턴 해줌
OK

> DEL name:1 name:2     # 앞서 사용한 예시 삭제
(integer) 2
> MSET name:1 "cafe" name:2 "mocha" name:3 "sejin"    # 여러개의 String key-value 한 번에 저장
OK
> MGET name:1 name:2 name:3                           # 여러개의 value값 조회
1) "cafe"
2) "mocha"
3) "sejin"
```

---

### ✅List

- 기본 명령어  
  **`LPUSH`** - 새 요소를 **헤드(처음)에 추가**  
  **`RPUSH`** - 새 요소를 **테일(끝)에 추가**  
  **`LPOP`** - **헤드(처음)의 요소를 삭제**하고 해당 요소를 리턴  
  **`RPOP`** - **테일(끝)의 요소를 삭제**하고 해당 요소를 리턴  
  **`LLEN`** - List의 **사이즈 리턴**  
  **`LMOVE`** - 요소를 한 목록에서 다른 목록으로 **원자적으로 이동**  
  **`LTRIM`** - List를 **지정된 범위로 줄임**

- List 특징  
  **순서가 있고 중복을 허용**  
  **처음이나 끝**에 **추가 및 삭제**가 **O(1)로 매우 빠름**  
  데이터가 많을때 **중간에 요소를 추가하거나 삭제할때** **비효율**적일수 있음  
  **대규모 데이터** 처리엔 **적합하지 않을 수 있음**

- redis-cli 예시

```bash
> LPUSH unit:add marine:1   # 처음(헤드)에 요소 추가 (marine:1)
(integer) 1
> LPUSH unit:add marine:2   # 처음(헤드)에 요소 추가 (marine:2, marine:1)
(integer) 2

> LRANGE unit:add 0 -1  # List 처음부터 끝까지 요소 조회
1) "marine:2"
2) "marine:1"

> RPOP unit:add   # 끝(테일)에 요소 삭제
"marine:1"
> RPOP unit:add
"marine:2"

> LLEN unit:add   # List 사이즈 리턴
(integer) 0

> RPUSH unit:add marine:1 marine:2 marine:3 marine:4 marine:5
(integer) 5
> LTRIM unit:add 0 2    # 0번 ~ 2번 인덱스까지 살리고 나머지 다 버림
OK
> LRANGE unit:add 0 -1  # 전체 조회하면 0번 ~ 2번 인덱스 요소가 있음
1) "marine:1"
2) "marine:2"
3) "marine:3"
```

---

### ✅Set

- 기본 명령어  
  **`SADD`** - Set에 **새 멤버를 추가**  
  **`SREM`** - Set에서 **지정된 멤버를 삭제**  
  **`SISMEMBER`** - Set에 **지정된 멤버가 있는지 리턴**  
  **`SINTER`** - 두 개 이상의 Set이 **공통으로 갖는 구성원 집합**(Ex: 교집합)을 리턴  
  **`SCARD`** - Set의 **사이즈**(a.k.a 카디널리티)를 리턴

- Set 특징  
  **순서가 없고 중복을 허용하지 않음**  
  **추가 및 삭제**가 **O(1)로 매우 빠름**  
  **합집합, 교집합, 차집합**과 같은 집합 연산을 지원  
  **대규모 데이터** 처리엔 **적합하지 않을 수 있음**

- redis-cli 예시

```bash
                                          # unit:terran:player1 Set
> SADD unit:terran:player1 scv:1          # scv:1 추가
(integer) 1                               # 성공 (1리턴)
> SADD unit:terran:player1 scv:1          # scv:1 추가
(integer) 0                               # 실패 (0리턴)
> SADD unit:terran:player1 scv:2 scv:3    # scv:2, scv:3 추가
(integer) 2
> SCARD unit:terran:player1               # Set 사이즈 리턴
(integer) 3

                                                  # unit:terran:player2 Set
> SADD unit:terran:player2 scv:1 scv:2            # scv:1, scv:2 추가
(integer) 2
> SINTER unit:terran:player1 unit:terran:player2  # player1, player2의 공통 Set 요소인 [scv:1, scv:2] 리턴
1) "scv:1"
2) "scv:2"
```

---

### ✅Sorted Set

- 기본 명령어  
  **`ZADD`** - Sorted Set에 **새 멤버와 연관된 점수를 추가** 단, 이미 **멤버가 존재**하는 경우 **점수가 업데이트** 됨  
  **`ZRANGE`** - **주어진 범위 내**에서 Sorted Set의 **멤버를 리턴**  
  **`ZRANK`** - **주어진 멤버의 순위를 리턴** (오름차순으로 되어 있다고 가정)  
  **`ZREVRANK`** - **주어진 멤버의 순위를 리턴** (내림차순으로 되어 있다고 가정)

- Sorted Set 특징  
  **순서가 있고 중복을 허용하지 않음**  
  **추가 및 삭제**가 **O(log(N))의 시간 복잡도를 가짐**  
  각 원소는 **고유한 점수(score)**와 연관되어 있음  
  각 원소의 **순위(rank)**를 조회할 수 있음

- redis-cli 예시

```bash
> ZADD player_scores 10 "mocha"           # Sorted Set(player_scores)에 mocha 10점 추가
(integer) 1
> ZADD player_scores 12 "sejin"           # Sorted Set(player_scores)에 sejin 12점 추가
(integer) 1
> ZADD player_scores 8 "cafe" 10 "latte" 6 "americano" 14 "coffee" # 한 번에 여러 요소 추가
(integer) 4

> ZRANGE player_scores 0 -1                   # 오름차순 전체 조회
1) "americano"
2) "cafe"
3) "latte"
4) "mocha"
5) "sejin"
6) "coffee"

> ZREVRANGE player_scores 0 -1                # 내림차순 전체 조회
1) "coffee"
2) "sejin"
3) "mocha"
4) "latte"
5) "cafe"
6) "americano"

> ZREVRANGE player_scores 0 -1 withscores     # 내림차순 전체 조회(with 스코어)
 1) "coffee"
 2) "14"
 3) "sejin"
 4) "12"
 5) "mocha"
 6) "10"
 7) "latte"
 8) "10"
 9) "cafe"
10) "8"
11) "americano"
12) "6"

> ZRANK player_scores "americano"             # 오름차순일때 americano 순위는 0번째 (1번이 아니라 0번부터)
(integer) 0
> ZREVRANK player_scores "americano"          # 내림차순일때 americano 순위는 5번째 (총 6명 중 6등임)
(integer) 5
```

---

### 📝마무리

이렇게 Redis에서 제공하는 다양한 data type에 대해 학습했다.  
각각의 특징을 잘 이해하고 있다면 필요한 상황에 맞는 data type을 선택할 수 있다.  
다음 포스팅에서는 Spring/java에서 redis를 이용해 간단한 랭킹 대시보드를 만들어 보자.

---

### 🌐References

- [Docker](https://www.docker.com/)
- [Redis](https://redis.io/docs/data-types/)

---
