1. 修改miner配置文件并重启
```
docker exec -it fil-2k-miner-miner mkdir /data/t01002/
docker exec -it fil-2k-miner-miner cp /var/lib/lotus-miner/config.toml /data/t01002/
vim /tmp/fil-2k-data/config.toml
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
vim /tmp/fil-2k-data/t01002/sectorstore.json
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
docker exec -it fil-2k-miner-miner lotus-miner storage attach --init --store /data/store/t01002
docker exec -it fil-2k-miner-miner lotus-miner storage list
```

3. 查看miner token
```
docker exec -it fil-2k-miner-miner lotus-miner auth api-info --perm admin
```

4. 启动worker
```
docker run -d -it \
  -e MINER_API_INFO="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBbGxvdyI6WyJyZWFkIiwid3JpdGUiLCJzaWduIiwiYWRtaW4iXX0.d43jQM92sDe-wJQRq6RkuP3Wk9r9lTgbYsgUlD2gwTk:/ip4/172.17.0.5/tcp/2345/http" \
  -e FIL_PROOFS_USE_GPU_COLUMN_BUILDER=1 \
  -e FIL_PROOFS_USE_GPU_TREE_BUILDER=1 \
  -e RUST_LOG=info \
  -v /minerCache/FIL_PROOFS_PARAMETER_CACHE:/var/tmp/filecoin-proof-parameters \
  --gpus all \
  --hostname fil-2k-miner-worker \
  --name fil-2k-miner-worker \
  -p 1238:3456 \
  he426100/lotus-worker:v1.13.1-dev \
  run
```

5. 质押
```
docker exec -it fil-2k-miner-miner lotus-miner sectors pledge
```