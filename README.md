# performanceImprovement84
Compare mysql 8.0 to mysql 8.4 with Thread Pool

## 0. 테스트 환경
오라클 클라우드(OCI) VM 2 대에 각각 mysql 8.0과 8.4 버전을 설치하고 각 서버마다 sysbench를 설치해서 부하를 다른서버에 주도록 구성

### MySQL 8.0 테스트 환경  
OS : OL 8.10  
CPU : ocpu 4   
Memory : 32 GB  
### MySQL 8.4 테스트 환경
OS : OL 9.6  
CPU : ocpu 4   
Memory : 32 GB  


## 1.sysbench 설치

sudo yum -y install make automake libtool pkgconfig libaio-devel

  
