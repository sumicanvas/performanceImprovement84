# MySQL 8.4의 성능 향상을 테스트
Compare mysql 8.0 to mysql 8.4 with Thread Pool

## 0. 테스트 환경
오라클 클라우드(OCI) VM 2 대에 각각 mysql 8.0과 8.4 버전을 설치하고 각 서버마다 sysbench를 설치해서 부하를 다른서버에 주도록 구성  
단, VM을 처음 생성 시  1 core, 16 GB 메모리로 생성 후 이후 edit를 통해서 4 core, 32 GB로 증가해서,  **innodb buffer pool size 및 read thread/write thread 등의 값은 1 core, 16 GB 에 맞춰져 있음.**  

### MySQL 8.0 테스트 환경  
OS : OL 8.10  
CPU : ocpu 4   
Memory : 32 GB  
### MySQL 8.4 테스트 환경
OS : OL 9.6  
CPU : ocpu 4   
Memory : 32 GB  


## 1.테스트 결과
아래 차트에서 보듯이 테스트환경이 열악했음에도 불구하고 8.4의 성능이 16% 정도 높게 나옴. 고객사의 
 <img width="1014" height="990" alt="image" src="https://github.com/user-attachments/assets/7740108e-e8ca-47a4-b811-d804b4bffb2b" />


## 1.sysbench 설치
아래 링크의 설치방법 참고  
https://github.com/sumicanvas/installsysbench

## 2.주요 mysql 설정 
8.0과 8.4 를 최대한 동일하게 설정했지만 버전 상 변경된 부분의 경우 반영해서 테스트함  

### 8.4 my.cnf
'''
[mysqld]  
innodb_dedicated_server=ON #8.4부터 기본적으로 활성화되어 있으며, 서버 메모리를 기반으로 innodb_buffer_pool_size, innodb_redo_log_capacity를 자동으로 계산

max-connections=2000  


'''

  <img width="1014" height="990" alt="image" src="https://github.com/user-attachments/assets/7740108e-e8ca-47a4-b811-d804b4bffb2b" />

