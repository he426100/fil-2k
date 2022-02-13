1. 构建镜像(官方的Dockerfile.lotus和scripts/docker-lotus-miner-entrypoint.sh有问题)
```
git clone https://github.com/he426100/fil-2k.git
git clone https://github.com/filecoin-project/lotus.git && cd lotus && git checkout v1.13.0
cp -r ../fil-2k/* ./
docker build -t lotus:v1.13.0-dev --target lotus -f Dockerfile.lotus-debug .
docker build -t lotus-seed:v1.13.0-dev --target lotus-seed -f Dockerfile.lotus-debug .
docker build -t lotus-miner:v1.13.0-dev --target lotus-miner -f Dockerfile.lotus-debug .
docker build -t lotus-worker:v1.13.0-dev --target lotus-worker -f Dockerfile.lotus-debug .
```

2. 创建或清空相关文件
```
# 首次
mkdir -p /workspace/tmp/fil-2k-data && sudo chown 532:532 /workspace/tmp/fil-2k-data
mkdir -p /workspace/tmp/fil-2k-lotus && sudo chown 532:532 /workspace/tmp/fil-2k-lotus
mkdir -p /workspace/tmp/fil-2k-lotus-miner && sudo chown 532:532 /workspace/tmp/fil-2k-lotus-miner
mkdir -p /var/tmp/filecoin-proof-parameters && sudo chown 532:532 /var/tmp/filecoin-proof-parameters
# 第二次
sudo rm -r /workspace/tmp/fil-2k-data/*
sudo rm -r /workspace/tmp/fil-2k-data/.genesis-sectors
sudo rm -r /workspace/tmp/fil-2k-lotus/*
sudo rm -r /workspace/tmp/fil-2k-lotus-miner/*
```

3. 预密封两个2KiB大小的扇区
```
docker run --rm \
  -v /workspace/tmp/fil-2k-data:/data \
  lotus-seed:v1.13.0-dev \
  --sector-dir /data/.genesis-sectors \
  pre-seal --sector-size=2KiB --num-sectors=2
```

4. 创建创世块
```
docker run --rm \
  -v /workspace/tmp/fil-2k-data:/data \
  lotus-seed:v1.13.0-dev \
  genesis new /data/localnet.json
```

5. 使用一些FIL为默认帐户提供资金
```
docker run --rm \
  -v /workspace/tmp/fil-2k-data:/data \
  lotus-seed:v1.13.0-dev \
  --sector-dir /data/.genesis-sectors \
  genesis add-miner /data/localnet.json /data/.genesis-sectors/pre-seal-t01000.json
```

6. 启动第一个节点
```
docker run -d -it \
  -e LOTUS_API_LISTENADDRESS="/ip4/0.0.0.0/tcp/1234/http" \
  -e DOCKER_LOTUS_IMPORT_SNAPSHOT="" \
  -v /workspace/tmp/fil-2k-data:/data \
  -v /workspace/tmp/fil-2k-lotus:/var/lib/lotus \
  -v /workspace/tmp/filecoin-proof-parameters:/var/tmp/filecoin-proof-parameters \
  --hostname fil-2k-master-lotus \
  --name fil-2k-master-lotus \
  -p 1234:1234 \
  lotus:v1.13.0-dev \
  daemon --lotus-make-genesis=/data/devgen.car --genesis-template=/data/localnet.json --bootstrap=false
```

7. 导入创世矿工密钥
```
docker exec -it fil-2k-master-lotus lotus wallet import --as-default /data/.genesis-sectors/pre-seal-t01000.key
docker exec -it fil-2k-master-lotus lotus wallet list
# 查看token
FULLNODE_API_INFO=$(docker exec -it fil-2k-master-lotus lotus auth api-info --perm admin | awk -F '=' '{print $2}'); \
  LOTUS_IP=$(docker exec -it fil-2k-master-lotus cat /etc/hosts | grep fil-2k-master-lotus | awk '{print $1}') ; \
  FULLNODE_API_INFO=${FULLNODE_API_INFO/0.0.0.0/$LOTUS_IP}; \
  echo $FULLNODE_API_INFO;
```

8. 初始化并运行创世矿工
```
# wsl2下不知道为啥不能用2345端口
docker run -d -it \
  -e LOTUS_API_LISTENADDRESS="/ip4/0.0.0.0/tcp/2345/http" \
  -e FULLNODE_API_INFO="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBbGxvdyI6WyJyZWFkIiwid3JpdGUiLCJzaWduIiwiYWRtaW4iXX0.iXcdXIZoRqs6_qw-c0d4xROczHFwQycMMZX7coH_scQ:/ip4/172.17.0.2/tcp/1234/http" \
  -e DOCKER_LOTUS_MINER_INIT=true \
  -e DOCKER_LOTUS_MINER_INIT_ARGS="--genesis-miner --actor=t01000 --sector-size=2KiB --pre-sealed-sectors=/data/.genesis-sectors --pre-sealed-metadata=/data/.genesis-sectors/pre-seal-t01000.json --nosync" \
  -v /workspace/tmp/fil-2k-data:/data \
  -v /workspace/tmp/fil-2k-lotus-miner:/var/lib/lotus-miner \
  -v /workspace/tmp/filecoin-proof-parameters:/var/tmp/filecoin-proof-parameters \
  --hostname fil-2k-master-miner \
  --name fil-2k-master-miner \
  -p 1235:2345 \
  lotus-miner:v1.13.0-dev \
  run --nosync
```

9. 查看创世矿工状态
```
docker exec -it fil-2k-master-miner lotus-miner info
```

10. fil-2k-master上获取创世节点的连接信息
```
docker exec -it fil-2k-master-lotus lotus net listen | grep "172"
```

11. 启动另一个节点
```
docker run -d -it \
  -e LOTUS_API_LISTENADDRESS="/ip4/0.0.0.0/tcp/1234/http" \
  -e DOCKER_LOTUS_IMPORT_SNAPSHOT="" \
  -v /workspace/tmp/fil-2k-data:/data \
  -v /workspace/tmp/filecoin-proof-parameters:/var/tmp/filecoin-proof-parameters \
  --hostname fil-2k-miner-lotus \
  --name fil-2k-miner-lotus \
  -p 1236:1234 \
  lotus:v1.13.0-dev \
  daemon --genesis=/data/devgen.car --bootstrap=false

# 查看token
FULLNODE_API_INFO=$(docker exec -it fil-2k-miner-lotus lotus auth api-info --perm admin | awk -F '=' '{print $2}'); \
  LOTUS_IP=$(docker exec -it fil-2k-miner-lotus cat /etc/hosts | grep fil-2k-miner-lotus | awk '{print $1}'); \
  FULLNODE_API_INFO=${FULLNODE_API_INFO/0.0.0.0/$LOTUS_IP}; \
  echo $FULLNODE_API_INFO;
```

12. fil-2k-miner上连接到创世节点
```
docker exec -it fil-2k-miner-lotus lotus net connect /ip4/172.17.0.2/tcp/34879/p2p/12D3KooWBSDyTJAtHF4F1wXd8L1hbcXR3H62oaV1DJjSBNcpjTVf
```

13. 查看当前节点的同步状态
```
docker exec -it fil-2k-miner-lotus lotus sync status
```

14. 创建BLS类型的钱包
```
docker exec -it fil-2k-miner-lotus lotus wallet new bls
```

15. 从创世节点处装100个FIL到当前钱包
```
docker exec -it fil-2k-master-lotus lotus send t3qfwutf7edp265gx5ynzjf5wfr4wt374ne25q6jjedatr7of7fzz3exiqjl4sn435cmg7x2qneerwyhf3mqeq 100
```

16. 检查余额
```
docker exec -it fil-2k-miner-lotus lotus wallet list
```

17. 初始化并运行新节点
```
docker run -d -it \
  -e LOTUS_API_LISTENADDRESS="/ip4/0.0.0.0/tcp/2345/http" \
  -e FULLNODE_API_INFO="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBbGxvdyI6WyJyZWFkIiwid3JpdGUiLCJzaWduIiwiYWRtaW4iXX0.Ru0z9bu6kL74h3UMyMGlesdXrsc2OzOSZDfkBycnhcM:/ip4/172.17.0.4/tcp/1234/http" \
  -e DOCKER_LOTUS_MINER_INIT=true \
  -e DOCKER_LOTUS_MINER_INIT_ARGS="--sector-size=2KiB" \
  -e RUST_LOG=info \
  -v /workspace/tmp/fil-2k-data:/data \
  -v /workspace/tmp/filecoin-proof-parameters:/var/tmp/filecoin-proof-parameters \
  --hostname fil-2k-miner-miner \
  --name fil-2k-miner-miner \
  -p 1237:2345 \
  lotus-miner:v1.13.0-dev \
  run
```

18. 查看新节点
```
docker exec -it fil-2k-miner-miner lotus-miner info
```

19. 修改miner配置文件并重启
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

20. miner挂载/data/store当作存储目录
```
docker exec -it fil-2k-miner-miner mkdir -p /data/store/t01002
docker exec -it fil-2k-miner-miner lotus-miner storage attach --init --store /data/store/t01002
docker exec -it fil-2k-miner-miner lotus-miner storage list
```

21. 查看miner token
```
MINER_API_INFO=$(docker exec -it fil-2k-miner-miner lotus-miner auth api-info --perm admin | awk -F '=' '{print $2}'); \
  MINER_IP=$(docker exec -it fil-2k-miner-miner cat /etc/hosts | grep fil-2k-miner-miner | awk '{print $1}') ; \
  MINER_API_INFO=${MINER_API_INFO/0.0.0.0/$MINER_IP}; \
  echo $MINER_API_INFO;
```

22. 启动worker
```
docker run -d -it \
  -e MINER_API_INFO="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBbGxvdyI6WyJyZWFkIiwid3JpdGUiLCJzaWduIiwiYWRtaW4iXX0.d43jQM92sDe-wJQRq6RkuP3Wk9r9lTgbYsgUlD2gwTk:/ip4/172.17.0.5/tcp/2345/http" \
  -e RUST_LOG=info \
  -v /workspace/tmp/fil-2k-data:/data \
  -v /workspace/tmp/filecoin-proof-parameters:/var/tmp/filecoin-proof-parameters \
  --hostname fil-2k-miner-worker \
  --name fil-2k-miner-worker \
  -p 1238:3456 \
  lotus-worker:v1.13.0-dev \
  run
```

23. worker挂载存储目录
```
docker exec -it fil-2k-miner-worker lotus-worker storage attach --store /data/store/t01002
docker exec -it fil-2k-miner-worker lotus-worker info
```

24. 查看worker
```
docker exec -it fil-2k-miner-miner lotus-miner sealing workers
docker exec -it fil-2k-miner-miner lotus-miner storage list
```

25. 质押
```
docker exec -it fil-2k-miner-miner lotus-miner sectors pledge
```

26. 查看扇区ID，获取扇区列表
```
docker exec -it fil-2k-miner-miner lotus-miner sectors list
```

27. 查看扇区信息
```
# 查看具体的某个扇区的状态（示例中显示的是扇区 0 的状态）
docker exec -it fil-2k-miner-miner lotus-miner sectors status 0
# 查看 0 号扇区的详细日志信息
docker exec -it fil-2k-miner-miner lotus-miner sectors status --log 0
```

### 感谢 [https://gitpod.io/](https://gitpod.io/)

参考资料  
[Developer network | Filecoin Docs](https://docs.filecoin.io/build/local-devnet/)  
[FileCoin Louts本地2K开发/测试环境部署指南](https://ansheng.me/filecoin-louts-local-2k-development-test-environment-deployment-guide/)
