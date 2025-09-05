# MySQL 8.4의 성능 향상을 테스트
Compare mysql 8.0 to mysql 8.4 with Thread Pool

## 0. 테스트 환경
오라클 클라우드(OCI) VM 2 대에 각각 mysql 8.0과 8.4 버전을 설치하고 각 서버마다 sysbench를 설치해서 부하를 다른서버에 주도록 구성  
단, VM을 처음 생성 시  1 core, 16 GB 메모리로 생성 후 이후 edit를 통해서 4 core, 32 GB로 증가해서,  **innodb buffer pool size 및 read thread/write thread 등의 값은 1 core, 16 GB 에 맞춰져 있음.**  

### 테스트는 sysbench를 이용해서 진행하며 OLTP_READ_WRITE(oltp_read_write.lua)로 진행했습니다.

### MySQL 8.0 테스트 환경  
OS : OL 8.10  
CPU : ocpu 4   
Memory : 32 GB  
mysql version : 8.0.42-commercial MySQL Enterprise Server

  
### MySQL 8.4 테스트 환경
OS : OL 9.6  
CPU : ocpu 4   
Memory : 32 GB  
mysql version :  8.4.5-commercial MySQL Enterprise Server


## 1.테스트 결과
결과부터 먼저 공유하자면 아래 차트에서 보듯이 테스트환경이 열악했음에도 불구하고 **8.4의 성능이 16% 정도 높게 나옴**. 고객사의 128 GB 메모리 32 core 환경에서는 **2배정도 높게** 결과가 나왔다고 전달받음. 사실 이번 테스트는 고객사지원을 위해서 진행한 건입니다.  
8.0 사용하고 계신 분들은 **내년 4월이면 mysql 8.0이 EOL되기때문에** 8.4로 업그레이드 하실 것을 권장합니다!!  

 <img width="507" height="445" alt="image" src="https://github.com/user-attachments/assets/7740108e-e8ca-47a4-b811-d804b4bffb2b" />


## 1.sysbench 설치
아래 링크의 설치방법 참고  
https://github.com/sumicanvas/installsysbench

## 2.주요 mysql 설정 
8.0과 8.4 를 최대한 동일하게 설정했지만 버전 상 기본값 등이 변경된 부분의 경우 반영해서 테스트함  

### 8.0 my.cnf  
innodb_dedicated_server=ON 을 한 경우 innodb redo log 사이즈가 커질 수 있다는 걸 감안해야 합니다.  
이 테스트 환경에서 redo log 관련 변수를 설정하지 않았을 때 18G까지 커지는 걸 확인했기때문에 가능하면 버전에 따라 innodb_redo_log_capacity 또는 innodb_log_files_in_group, innodb_log_file_size 값을 설정해 주기를 권장함.
**아래는 이번 테스트환경의 파라미터 예시이며 각자 환경에 따라 모든 파라미터는 조정되어야 함. 이번 테스트를 진행한 vm의 경우 스펙이 낮아서, 운영환경인 경우 각 서버의 cpu, memory에 따라 해당 값들은 반드시 튜닝되어야 함.**
```
[mysqld]  
innodb_dedicated_server=ON  
max-connections=2000  
  
innodb_buffer_pool_size=8G  
innodb_buffer_pool_instances=16  
innodb_page_cleaners=16  
innodb_flush_method=O_DIRECT  
innodb_io_capacity=10000  
innodb_io_capacity_max=20000  
  
innodb_read_io_threads = 8  
innodb_write_io_threads = 4  
innodb_log_files_in_group=2
innodb_log_file_size=256M 
  
# Plugin load (MySQL Enterprise ThreadPool)  
plugin-load-add=thread_pool.so  
thread_pool_size=16  
thread_pool_algorithm=1  
thread_pool_max_transactions_limit=512  
thread_pool_query_threads_per_group = 2  
thread_pool_stall_limit=6

```

### 8.4 my.cnf

```
[mysqld]  
innodb_dedicated_server=ON  #8.4부터 기본적으로 활성화되어 있으며, 서버 메모리를 기반으로 innodb_buffer_pool_size, innodb_redo_log_capacity를 자동으로 계산

max-connections=2000  
  
innodb_buffer_pool_size=8G  
innodb_buffer_pool_instances=16  
innodb_page_cleaners=16    
innodb_flush_method=O_DIRECT  
innodb_io_capacity=10000  
innodb_io_capacity_max=20000  
  
innodb_read_io_threads = 8  
innodb_write_io_threads = 4  
innodb_redo_log_capacity=1G  
  
# Plugin load (MySQL Enterprise ThreadPool)  
plugin-load-add=thread_pool.so  
thread_pool_size=16  
thread_pool_algorithm=1  
thread_pool_max_transactions_limit=512  
thread_pool_query_threads_per_group = 2  
thread_pool_stall_limit=6

```
  
## 3. 테스트 사용자 생성 : 8.4에서 Thread Pool 을 함께 테스트할 경우 반드시 일반 사용자를 생성 후 테스트!!  
**mysql 8.4** 에서 권한이 세분화되면서 변경된 사항이 있습니다. **root나 SUPPER 권한을 가진 사용자로 Thread Pool을 테스트하시면 안됩니다!**  
root와 SUPPER 권한을 가진 사용자(ex.admin)의 경우 TP_CONNECTION_ADMIN privilege 를 가지기때문에 일정 수 이상으로 Active Thread가 생성되지 않아 성능이 제대로 나오지 않습니다.   
8.4에서 변경된 부분으로 8.0 에서는 문제없이 동작됩니다. 하지만 가능한 application 사용자가 가져야 하는 권한을 부여한 일반사용자를 생성 후 사용하실 것을 권장합니다.  
  
```
DROP USER '테스트용 일반사용자'@'%';

CREATE USER '테스트용 일반사용자'@'%' IDENTIFIED WITH caching_sha2_password BY '비밀번호';

GRANT SELECT, INSERT, DELETE, UPDATE, CREATE, DROP, PROCESS, USAGE, INDEX ON *.* TO '테스트용 일반사용자'@'%';
GRANT SHOW DATABASES ON *.* TO '테스트용 일반사용자'@'%';

-- Enforce SSL only connections (if required)
ALTER USER '테스트용 일반사용자'@'%' REQUIRE SSL;  
```

## 4.sysbench 테스트 명령어
### 4-1 prepare
미리 테스트용 데이터베이스(ex.sysbench)를 생성해둬야 함
sysbench /usr/sysbench/share/sysbench/oltp_read_write.lua    --table-size=100000   --tables=5  --time=100   --report-interval=10  --rand-type=uniform --db-driver=mysql   --mysql-host=<8.0 또는 8.4서버IP>   --mysql-user='테스트사용자'   --mysql-password='비밀번호'   --mysql-port=3306 --mysql-db=<테스트db명>  --threads=thread수  prepare

    
### 4-2 run (oltp_read_write.lua)
테스트는 thread를 64, 100, 200, 300, 500 으로 변경하며 테스트 진행
sysbench /usr/sysbench/share/sysbench/oltp_read_write.lua    --table-size=1000000   --tables=5  --time=100   --report-interval=10  --rand-type=uniform --db-driver=mysql   --mysql-host=<8.0 또는 8.4서버IP>  --mysql-user='테스트사용자'   --mysql-password='비밀번호'   --mysql-port=3306 --mysql-db=<테스트db명>  --threads=thread수  run  


  
