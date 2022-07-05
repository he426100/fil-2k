1. 修改miner配置文件并重启
```
docker exec -it fil-2k-miner-miner mkdir /data/t01002/
docker exec -it fil-2k-miner-miner cp /var/lib/lotus-miner/config.toml /data/t01002/
sudo vim /tmp/fil-2k-data/t01002/config.toml
```
```
[Storage]
  # env var: LOTUS_STORAGE_PARALLELFETCHLIMIT
  #ParallelFetchLimit = 10

  # env var: LOTUS_STORAGE_ALLOWADDPIECE
  AllowAddPiece = false

  # env var: LOTUS_STORAGE_ALLOWPRECOMMIT1
  AllowPreCommit1 = false

  # env var: LOTUS_STORAGE_ALLOWPRECOMMIT2
  AllowPreCommit2 = false

  # env var: LOTUS_STORAGE_ALLOWCOMMIT
  AllowCommit = false

  # env var: LOTUS_STORAGE_ALLOWUNSEAL
  AllowUnseal = false

  # env var: LOTUS_STORAGE_RESOURCEFILTERING
  #ResourceFiltering = "hardware"
```
```
docker exec -it fil-2k-miner-miner cp /var/lib/lotus-miner/sectorstore.json /data/t01002/
sudo vim /tmp/fil-2k-data/t01002/sectorstore.json
```
```
{
  "ID": "f7328656-4888-4dd7-bed5-aa2a1aa9a64f",
  "Weight": 10,
  "CanSeal": true,
  "CanStore": false,
  "MaxStorage": 0
}
```
重启miner
```
docker exec -it fil-2k-miner-miner cp /data/t01002/config.toml /var/lib/lotus-miner/
docker exec -it fil-2k-miner-miner cp /data/t01002/sectorstore.json /var/lib/lotus-miner/
docker restart fil-2k-miner-miner
```

2. miner挂载/data/store当作存储目录
```
docker exec -it fil-2k-miner-miner mkdir -p /data/store/t01002
docker exec -it fil-2k-miner-miner lotus-miner storage attach --init --store /data/store/t01002
docker exec -it fil-2k-miner-miner lotus-miner storage list
```

3. 查看miner token
```
MINER_API_INFO=$(docker exec -it fil-2k-miner-miner lotus-miner auth api-info --perm admin | awk -F '=' '{print $2}'); \
  MINER_IP=$(docker exec -it fil-2k-miner-miner cat /etc/hosts | grep fil-2k-miner-miner | awk '{print $1}') ; \
  MINER_API_INFO=$(echo ${MINER_API_INFO/0.0.0.0/$MINER_IP} | sed -e 's/\r//g'); \
  echo $MINER_API_INFO;
```

4. 启动worker
```
docker run -d -it \
  -e MINER_API_INFO="$MINER_API_INFO" \
  -e FIL_PROOFS_USE_GPU_COLUMN_BUILDER=1 \
  -e FIL_PROOFS_USE_GPU_TREE_BUILDER=1 \
  -e RUST_LOG=info \
  -v /tmp/fil-2k-data:/data \
  -v /var/tmp/filecoin-proof-parameters:/var/tmp/filecoin-proof-parameters \
  --gpus all \
  --hostname fil-2k-miner-worker \
  --name fil-2k-miner-worker \
  -p 1238:3456 \
  lotus-worker:v1.16.0-dev \
  run
```

5. worker挂载存储目录
```
docker exec -it fil-2k-miner-worker lotus-worker storage attach --store /data/store/t01002
docker exec -it fil-2k-miner-worker lotus-worker info
```

6. 查看worker
```
docker exec -it fil-2k-miner-miner lotus-miner sealing workers
docker exec -it fil-2k-miner-miner lotus-miner storage list
```

7. 质押
```
docker exec -it fil-2k-miner-miner lotus-miner sectors pledge
```
