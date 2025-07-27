docker run -d \

  --name hbase \

  --hostname hbase-master \

  -p 16010:16010 -p 9090:9090 -p 2181:2181 \

  --platform linux/amd64 \

  dajobe/hbase





docker exec -it hbase bash



hbase shell



# 테이블 생성

create 'test', 'cf'



# 데이터 삽입

put 'test', 'row1', 'cf:name', 'Alice'

put 'test', 'row2', 'cf:name', 'Bob'



# 데이터 조회

get 'test', 'row1'



# 테이블 전체 스캔

scan 'test'



# 테이블 목록

list



# 테이블 삭제

disable 'test'

drop 'test'
