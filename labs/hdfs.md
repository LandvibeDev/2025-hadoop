```
function de() {
    # Docker 컨테이너 선택
    local CONTAINER=$(docker ps --format '{{.ID}} {{.Names}}' | fzf --height 40% --layout=reverse --info=inline --prompt "Select container: ")

    if [ -z "$CONTAINER" ]; then
        echo "No container selected. Exiting."
        return 1
    fi

    # 선택된 컨테이너 ID 추출
    local CONTAINER_ID=$(echo $CONTAINER | awk '{print $1}')

    # 사용 가능한 쉘 선택
    local SHELL=$(echo -e "bash\nsh\nzsh\nash" | fzf --height 40% --layout=reverse --info=inline --prompt "Select shell: ")

    if [ -z "$SHELL" ]; then
        echo "No shell selected. Exiting."
        return 1
    fi

    # 선택된 컨테이너와 쉘로 docker exec 실행
    docker exec -it $CONTAINER_ID $SHELL
}
```

./docker-compose.yaml
```yaml
services:
  namenode:
    image: apache/hadoop:3.3.6
    hostname: namenode
    command: ["hdfs", "namenode"]
    ports:
      - "9870:9870"        # NameNode UI
    env_file:
      - ./config
    environment:
      ENSURE_NAMENODE_DIR: "/tmp/hadoop-root/dfs/name"
    # platform: "linux/arm64/v8"   # (필요시) ARM 강제. 문제가 생기면 주석 해제
    # platform: "linux/amd64"      # (대안) x86 에뮬로 실행해야 할 때

  datanode:
    image: apache/hadoop:3.3.6
    command: ["hdfs", "datanode"]
    env_file:
      - ./config
    depends_on:
      - namenode
    # 주의: datanode UI(9864) 등은 스케일링 시 포트 충돌 나므로 노출하지 않음

  resourcemanager:
    image: apache/hadoop:3.3.6
    hostname: resourcemanager
    command: ["yarn", "resourcemanager"]
    ports:
      - "8088:8088"        # YARN RM UI
    env_file:
      - ./config
    depends_on:
      - namenode

  nodemanager:
    image: apache/hadoop:3.3.6
    command: ["yarn", "nodemanager"]
    env_file:
      - ./config
    depends_on:
      - resourcemanager

```

./config
```
# 기본 경로
HADOOP_HOME=/opt/hadoop

# -------- core-site.xml --------
CORE-SITE.XML_fs.defaultFS=hdfs://namenode:8020

# -------- hdfs-site.xml --------
HDFS-SITE.XML_dfs.namenode.rpc-address=namenode:8020
HDFS-SITE.XML_dfs.replication=3
HDFS-SITE.XML_dfs.namenode.stale.datanode.interval=10000         # 10초 후 Stale
HDFS-SITE.XML_dfs.namenode.heartbeat.recheck-interval=30000      # 30초 후 Dead 판정 시도
HDFS-SITE.XML_dfs.heartbeat.interval=1                            # DN 하트비트 1초

# 로컬 단일 호스트 도커 네트워크에서 편의용(옵션)
# HDFS-SITE.XML_dfs.client.use.datanode.hostname=true

# -------- yarn-site.xml --------
YARN-SITE.XML_yarn.resourcemanager.hostname=resourcemanager
YARN-SITE.XML_yarn.nodemanager.pmem-check-enabled=false
YARN-SITE.XML_yarn.nodemanager.vmem-check-enabled=false

# 작은 리소스로도 3개 NM가 뜨도록 자원 한도 축소
YARN-SITE.XML_yarn.nodemanager.resource.memory-mb=2048
YARN-SITE.XML_yarn.nodemanager.resource.cpu-vcores=2
YARN-SITE.XML_yarn.scheduler.minimum-allocation-mb=256
YARN-SITE.XML_yarn.scheduler.maximum-allocation-mb=1536
YARN-SITE.XML_yarn.nodemanager.aux-services=mapreduce_shuffle

# -------- mapred-site.xml --------
MAPRED-SITE.XML_mapreduce.framework.name=yarn
MAPRED-SITE.XML_mapreduce.map.memory.mb=256
MAPRED-SITE.XML_mapreduce.reduce.memory.mb=256
MAPRED-SITE.XML_mapreduce.map.java.opts=-Xmx192m
MAPRED-SITE.XML_mapreduce.reduce.java.opts=-Xmx192m
MAPRED-SITE.XML_yarn.app.mapreduce.am.resource.mb=256
MAPRED-SITE.XML_yarn.app.mapreduce.am.command-opts=-Xmx192m
MAPRED-SITE.XML_mapreduce.framework.name=yarn
MAPRED-SITE.XML_yarn.app.mapreduce.am.env=HADOOP_MAPRED_HOME=/opt/hadoop
MAPRED-SITE.XML_mapreduce.map.env=HADOOP_MAPRED_HOME=/opt/hadoop
MAPRED-SITE.XML_mapreduce.reduce.env=HADOOP_MAPRED_HOME=/opt/hadoop

# -------- capacity-scheduler.xml (필수 최소값들) --------
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.maximum-applications=10000
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.resource-calculator=org.apache.hadoop.yarn.util.resource.DefaultResourceCalculator
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.root.queues=default
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.root.default.capacity=100
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.root.default.maximum-capacity=100
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.root.default.state=RUNNING
```

```bash
# 이미지 받기
docker compose pull

# 3개의 DataNode & 3개의 NodeManager로 기동
docker compose up -d --scale datanode=3 --scale nodemanager=3
```


# 하둡 CLI 실습 가이드 (3시간 커리큘럼)

**목표**: HDFS 기본 조작, 복제/복구 동작 관찰, 스냅샷/쿼터, YARN 잡 실행과 로그 확인까지 한 번에 익히기.  
**사전 준비**: docker-compose로 namenode, resourcemanager, datanode×3, nodemanager×3 실행 완료.

---

## 0. 환경 점검 (10분)
네임노드 컨테이너로 진입 후 상태를 점검합니다.
```bash
hn
hdfs dfsadmin -report        # DataNode 수/용량/블록 통계
yarn node -list -all         # NodeManager 목록
hdfs dfs -ls /               # 기본 루트 디렉토리 확인
```

## 1. HDFS 기초 명령 (35분)
개인 작업 디렉토리를 만들고(없으면) 기본 FS 셸을 익힙니다.
```bash
# 1) 작업 디렉토리 준비
whoami
hdfs dfs -mkdir -p /user/hadoop
hdfs dfs -chown hadoop /user/hadoop
hdfs dfs -ls /

# 2) 파일 올리기/보기
dd if=/dev/urandom of=bigfile_100m.bin bs=1M count=100
hdfs dfs -put bigfile_100m.bin /user/hadoop/
hdfs dfs -ls -h /user/hadoop/
hdfs dfs -du -h /user/hadoop/bigfile_100m.bin
hdfs dfs -stat %r /user/hadoop/bigfile_100m.bin    # 복제계수 확인
hdfs dfs -cat /user/hadoop/bigfile_100m.bin | head -c 64 | hexdump -C
hdfs dfs -checksum /user/hadoop/bigfile_100m.bin

# 3) 복사/이동/삭제/권한
hdfs dfs -cp /user/hadoop/bigfile_100m.bin /user/hadoop/copy.bin
hdfs dfs -mv /user/hadoop/copy.bin /user/hadoop/moved.bin
hdfs dfs -chmod 640 /user/hadoop/moved.bin
hdfs dfs -rm -r -skipTrash /user/hadoop/moved.bin
```

## 2. 복제/내고장성 관찰 (30분)
복제계수 변경과 DataNode 중지/복구로 HDFS의 자가 치유를 체험합니다.
```bash
# 0) 대용량 파일 추가
dd if=/dev/urandom of=bigfile_2g.bin bs=1M count=2000
hdfs dfs -put bigfile_2g.bin /user/hadoop/

# 1) 복제계수 변경
hdfs dfs -setrep -w 2 /user/hadoop/bigfile_2g.bin
hdfs dfs -stat %r /user/hadoop/bigfile_2g.bin

# 2) DataNode 하나 정지(호스트 쉘에서)
docker compose ps datanode
docker stop <datanode container id>

# 3) 네임노드에서 상태 확인
hdfs dfs -setrep -w 4 /user/hadoop/bigfile_2g.bin
hdfs dfsadmin -report | egrep 'Live datanodes|Under replicated blocks'
hdfs fsck / -files -blocks -locations | head -200

# 4) DataNode 재기동 후 복구 확인(호스트)
docker start $(docker compose ps -q datanode | head -1)

# 5) 복제계수 원복
hn
hdfs dfs -setrep -w 3 /user/hadoop/bigfile_100m.bin
```

## 3. SafeMode & fsck (10분)
SafeMode 동작을 간단 확인하고 fsck로 파일/블록/위치를 점검합니다.
```bash
hdfs dfsadmin -safemode get
hdfs dfsadmin -safemode enter
hdfs dfsadmin -safemode leave
hdfs fsck / -files -blocks -locations | sed -n '1,200p'
```

## 4. 스냅샷과 복원 (20분)
지정 디렉토리에서 스냅샷 생성/비교/복원을 실습합니다.
```bash
# 스냅샷 허용 & 생성
hdfs dfsadmin -allowSnapshot /user/hadoop
hdfs dfs -createSnapshot /user/hadoop s1
hdfs dfs -ls -h /user/hadoop/.snapshot
hdfs dfs -ls -h /user/hadoop/.snapshot/s1

# 파일 수정 후 두 번째 스냅샷
dd if=/dev/urandom of=bigfile_50m.bin bs=1M count=50
hdfs dfs -put -f bigfile_50m.bin /user/hadoop/bigfile_100m.bin
hdfs dfs -createSnapshot /user/hadoop s2
hdfs dfs -ls -h /user/hadoop/.snapshot
hdfs dfs -ls -h /user/hadoop/.snapshot/s2

# 스냅샷 비교 & 복원
hdfs snapshotDiff /user/hadoop s1 s2
hdfs dfs -cp /user/hadoop/.snapshot/s1/bigfile_100m.bin /user/hadoop/restored.bin
hdfs dfs -ls -h /user/hadoop

# 스냅샷 삭제 & 스냅샷 비허용
hdfs dfs -deleteSnapshot /user/hadoop s1
hdfs dfs -deleteSnapshot /user/hadoop s2
hdfs dfsadmin -disallowSnapshot /user/hadoop
```

## 5. 디렉토리/용량 Quota (10분)
폴더별 파일/공간 쿼터를 설정하고 초과 시 동작을 관찰합니다.
```bash
hdfs dfsadmin -setQuota 1000 /user/hadoop
hdfs dfsadmin -setSpaceQuota 200m /user/hadoop

# 용량 초과 유도(실패 메시지 확인)
dd if=/dev/urandom of=bigfile_300m.bin bs=1M count=300
hdfs dfs -put bigfile_300m.bin /user/hadoop/    # 공간 부족 오류 기대

# 현황 확인
hdfs dfs -count -q -h /user/hadoop
```

## 6. Balancer 사용 (10분)
블록 분포가 고르지 않을 때 Balancer를 실행해 균형을 맞춥니다.
```bash
hdfs balancer -threshold 1
# 진행 상황은 NameNode UI(9870)와 로그에서 확인
```

## 7. YARN으로 예제 잡 실행 (30분)
Pi/WordCount 같은 예제 잡을 실행하고 상태/로그를 확인합니다.
```bash
# 노드/큐/애플리케이션 확인
yarn node -list -all
yarn queue -list

# Pi 계산 예제
yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar pi 5 5

# WordCount 예제
echo -e "a b a b c\na a b b c c" > words.txt
hdfs dfs -put -f words.txt /user/hadoop/words.txt
yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar wordcount /user/hadoop/words.txt /user/hadoop/wc_out
hdfs dfs -cat /user/hadoop/wc_out/part-r-00000

# 컨테이너 종료(호스트)
docker compose down
```
