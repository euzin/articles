# RDS PGSQL 성능시험을 위한 sysbench 설치 및 테스트 스크립트
#### Sysbench  

> sysbench 는 MYSQL AB 사의 성능팀 내부 프로젝트로 시작되었다고 한다. 
> 처음에는 단일 c 소스였고, 그이후에도 큰 변화 없이 소소한 리팩토링을 거듭하다가 
> 최근에는 1.0 으로 업그레이드 되었다. (sysbench 0.5 부터 oltp test 옵션이 lua script 로 분리 되는 등.)
> https://github.com/akopytov/sysbench 깃헙에는 1.0 으로 올라와 있고 
> [ 개발자 블로그 ](http://kaamos.me/blog//2016/03/08/towards-sysbench-1.0-history.html) 앞으로 열심히 업데이트 한다는 글이 있다.  

- db 관련 테스트 외에 아래와 같은 테스트 옵션도 쓸만함.

```
<test opions>
fileio – File I/O test
cpu – CPU performance test
memory – Memory functions speed test
threads – Threads subsystem performance test
mutex – Mutex performance test
```

#### Install for pgsql
```
yum install postgresql-devel 
git clone https://github.com/akopytov/sysbench.git
cd sysbench
autogen.sh
/configure --with-pgsql --with-pgsql-includes=/usr/include/pgsql92 --with-pgsql-libs=/usr/lib64/pgsql92
make; make install
```

#### Test workload  
- 논리적인 sql 은 well tuned 로 가정 하고 서버 스팩과 io subsystem 의 성능을 비교 검증하는 용도로 사용하던 oltp worklaod  
- read io 위주 테스트와  write io 위주 테스트로 구성 
- read/write 비율은 --oltp-index-update, --oltp-non-index-update 옵션으로 조정한다. 
- test shell : 

```
#!/bin/bash

SYSBENCH=/root/db/sysbench/sysbench/sysbench
DB_USER=ㄸㄸㄸ
DB_PASS=ㄸㄸㄸ
DB_SOCK=
DB_NAME=sbtest
PGSQL_HOST=ㄸㄸㄸ
PGSQL_PORT=ㄸㄸㄸ
OLTP=oltp.lua

func_oltp_test_read_remote()
{
   NOW=`date '+%Y%m%d_%H%M'`
   $SYSBENCH --db-driver=pgsql --pgsql-host=$PGSQL_HOST --pgsql-port=$PGSQL_PORT --pgsql-user=$DB_USER --pgsql-password=$DB_PASS --pgsql-db=$DB_NAME \
             --forced-shutdown=5 \
             --rand-type=uniform \
             --num-threads=$CONN_THREADS \
             --max-requests=$MAX_REQUEST \
             --max-time=$MAX_TIME \
             --oltp_tables_count=$NUMBER_OF_TABLE \
             --test=$OLTP \
             --oltp-table-size=$OLTP_TABLE_SIZE \
             --oltp-test-mode=complex  \
             --oltp-range-size=$OLTP_RANGE_SIZE \
             --oltp-index-updates=1 \
             --oltp-non-index-updates=1 \
             run > test.r.$NOW
   cat test.r.$NOW
   sleep $SLEEP
   echo ""
}

func_oltp_test_write_remote()
{
   NOW=`date '+%Y%m%d_%H%M'`
   $SYSBENCH --db-driver=pgsql --pgsql-host=$PGSQL_HOST --pgsql-port=$PGSQL_PORT --pgsql-user=$DB_USER --pgsql-password=$DB_PASS --pgsql-db=$DB_NAME\
             --forced-shutdown=5 \
             --rand-type=uniform \
             --num-threads=$CONN_THREADS \
             --max-requests=$MAX_REQUEST \
             --max-time=$MAX_TIME \
             --oltp_tables_count=$NUMBER_OF_TABLE \
             --test=$OLTP \
             --oltp-table-size=$OLTP_TABLE_SIZE \
             --oltp-test-mode=complex  \
             --oltp-range-size=$OLTP_RANGE_SIZE \
             --oltp-index-updates=25 \
             --oltp-non-index-updates=25 \
             run > test.w.$NOW
   cat test.w.$NOW
   sleep $SLEEP
   echo ""
}

func_cleanup_remote()
{
   $SYSBENCH --db-driver=pgsql --test=oltp \
             --pgsql-host=$PGSQL_HOST \
             --pgsql-port=$PGSQL_PORT \
             --pgsql-user=$DB_USER \
             --pgsql-password=$DB_PASS \
             --oltp-tables-count=16 \
             cleanup
}

func_prepare_remote()
{
   $SYSBENCH --db-driver=pgsql --test=parallel_prepare.lua \
             --pgsql-host=$PGSQL_HOST \
             --pgsql-port=$PGSQL_PORT \
             --pgsql-user=$DB_USER \
             --pgsql-password=$DB_PASS \
             --oltp-table-size=$OLTP_TABLE_SIZE \
             --oltp-tables-count=$NUMBER_OF_TABLE \
             --num-threads=16 \
             run
}

func_oltp_conn()
{
    CONN_THREADS=32;func_oltp_test_read_remote
    sleep $SLEEP
    CONN_THREADS=32;func_oltp_test_write_remote
}
echo "##################### oltp test #######################"
echo "START :" `date`

SLEEP=60
TEST_COUNT=1
NUMBER_OF_TABLE=16
NUMBER_OF_QUERY=10
OLTP_RANGE_SIZE=100
OLTP_TABLE_SIZE=300000
MAX_REQUEST=0
MAX_TIME=300

func_cleanup_remote
func_prepare_remote
sleep 15
func_oltp_conn
echo "End:" `date`
```


