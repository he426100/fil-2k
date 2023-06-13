1. 构建镜像
```
git clone https://github.com/filecoin-project/lotus.git && cd lotus && git checkout v1.23.0
wget https://raw.githubusercontent.com/he426100/fil-2k/master/Dockerfile.lotus-debug
docker build -t lotus:v1.23.0-dev --target lotus -f Dockerfile.lotus-debug .
docker build -t lotus-seed:v1.23.0-dev --target lotus-seed -f Dockerfile.lotus-debug .
docker build -t lotus-miner:v1.23.0-dev --target lotus-miner -f Dockerfile.lotus-debug .
docker build -t lotus-worker:v1.23.0-dev --target lotus-worker -f Dockerfile.lotus-debug .
```

2. 创建或清空相关文件
```
# 首次
mkdir -p /tmp/fil-2k-data && sudo chown 532:532 /tmp/fil-2k-data
mkdir -p /tmp/fil-2k-lotus && sudo chown 532:532 /tmp/fil-2k-lotus
mkdir -p /tmp/fil-2k-lotus-miner && sudo chown 532:532 /tmp/fil-2k-lotus-miner
mkdir -p /var/tmp/filecoin-proof-parameters && sudo chown 532:532 /var/tmp/filecoin-proof-parameters
# 第二次
sudo rm -r /tmp/fil-2k-data/*
sudo rm -r /tmp/fil-2k-data/.genesis-sectors
sudo rm -r /tmp/fil-2k-lotus/*
sudo rm -r /tmp/fil-2k-lotus-miner/*
```

3. 预密封两个2KiB大小的扇区
```
docker run --rm \
  -v /tmp/fil-2k-data:/data \
  lotus-seed:v1.23.0-dev \
  --sector-dir /data/.genesis-sectors \
  pre-seal --sector-size=2KiB --num-sectors=2
```

4. 创建创世块
```
docker run --rm \
  -v /tmp/fil-2k-data:/data \
  lotus-seed:v1.23.0-dev \
  genesis new /data/localnet.json
```

5. 使用一些FIL为默认帐户提供资金
```
docker run --rm \
  -v /tmp/fil-2k-data:/data \
  lotus-seed:v1.23.0-dev \
  --sector-dir /data/.genesis-sectors \
  genesis add-miner /data/localnet.json /data/.genesis-sectors/pre-seal-t01000.json
```

6. 启动第一个节点
```
docker run -d -it \
  -e LOTUS_API_LISTENADDRESS="/ip4/0.0.0.0/tcp/1234/http" \
  -e DOCKER_LOTUS_IMPORT_SNAPSHOT="" \
  -v /tmp/fil-2k-data:/data \
  -v /var/tmp/filecoin-proof-parameters:/var/tmp/filecoin-proof-parameters \
  --hostname fil-2k-master-lotus \
  --name fil-2k-master-lotus \
  -p 1234:1234 \
  lotus:v1.23.0-dev \
  daemon --lotus-make-genesis=/data/devgen.car --genesis-template=/data/localnet.json --bootstrap=false
```

7. 导入创世矿工密钥
```
docker exec -it fil-2k-master-lotus lotus wallet import --as-default /data/.genesis-sectors/pre-seal-t01000.key
docker exec -it fil-2k-master-lotus lotus wallet list
# 查看token
FULLNODE_API_INFO=$(docker exec -it fil-2k-master-lotus lotus auth api-info --perm admin | awk -F '=' '{print $2}'); \
  LOTUS_IP=$(docker exec -it fil-2k-master-lotus cat /etc/hosts | grep fil-2k-master-lotus | awk '{print $1}'); \
  FULLNODE_API_INFO=$(echo ${FULLNODE_API_INFO/0.0.0.0/$LOTUS_IP} | sed -e 's/\r//g'); \
  echo $FULLNODE_API_INFO;
```

8. 初始化并运行创世矿工
```
# wsl2下不知道为啥不能用2345端口
docker run -d -it \
  -e LOTUS_API_LISTENADDRESS="/ip4/0.0.0.0/tcp/2345/http" \
  -e FULLNODE_API_INFO="$FULLNODE_API_INFO" \
  -e DOCKER_LOTUS_MINER_INIT=true \
  -e DOCKER_LOTUS_MINER_INIT_ARGS="--genesis-miner --actor=t01000 --sector-size=2KiB --pre-sealed-sectors=/data/.genesis-sectors --pre-sealed-metadata=/data/.genesis-sectors/pre-seal-t01000.json --nosync" \
  -v /tmp/fil-2k-data:/data \
  -v /var/tmp/filecoin-proof-parameters:/var/tmp/filecoin-proof-parameters \
  --hostname fil-2k-master-miner \
  --name fil-2k-master-miner \
  -p 1235:2345 \
  lotus-miner:v1.23.0-dev \
  run --nosync
```

9. 查看创世矿工状态
```
docker exec -it fil-2k-master-miner lotus-miner info
```

10. 启动另一个节点
```
docker run -d -it \
  -e LOTUS_API_LISTENADDRESS="/ip4/0.0.0.0/tcp/1234/http" \
  -e DOCKER_LOTUS_IMPORT_SNAPSHOT="" \
  -v /tmp/fil-2k-data:/data \
  -v /tmp/fil-2k-lotus:/var/lib/lotus \
  -v /var/tmp/filecoin-proof-parameters:/var/tmp/filecoin-proof-parameters \
  --hostname fil-2k-miner-lotus \
  --name fil-2k-miner-lotus \
  -p 1236:1234 \
  lotus:v1.23.0-dev \
  daemon --genesis=/data/devgen.car --bootstrap=false

# 查看token
FULLNODE_API_INFO=$(docker exec -it fil-2k-miner-lotus lotus auth api-info --perm admin | awk -F '=' '{print $2}'); \
  LOTUS_IP=$(docker exec -it fil-2k-miner-lotus cat /etc/hosts | grep fil-2k-miner-lotus | awk '{print $1}'); \
  FULLNODE_API_INFO=$(echo ${FULLNODE_API_INFO/0.0.0.0/$LOTUS_IP} | sed -e 's/\r//g'); \
  echo $FULLNODE_API_INFO;
```

11. fil-2k-miner上连接到创世节点
```
docker exec -it fil-2k-miner-lotus lotus net connect $(docker exec -it fil-2k-master-lotus lotus net listen | grep "172" | sed -e 's/\r//g')
```

12. 查看当前节点的同步状态
```
docker exec -it fil-2k-miner-lotus lotus sync status
```

13. 创建BLS类型的钱包并从创世节点处装100个FIL到当前钱包
```
docker exec -it fil-2k-master-lotus lotus send $(docker exec -it fil-2k-miner-lotus lotus wallet new bls | sed -e 's/\r//g') 100
```

14. 检查余额
```
docker exec -it fil-2k-miner-lotus lotus wallet list
```

15. 初始化并运行新节点
```
docker run -d -it \
  -e LOTUS_API_LISTENADDRESS="/ip4/0.0.0.0/tcp/2345/http" \
  -e FULLNODE_API_INFO="$FULLNODE_API_INFO" \
  -e DOCKER_LOTUS_MINER_INIT=true \
  -e DOCKER_LOTUS_MINER_INIT_ARGS="--sector-size=2KiB" \
  -e RUST_LOG=info \
  -v /tmp/fil-2k-data:/data \
  -v /tmp/fil-2k-lotus-miner:/var/lib/lotus-miner \
  -v /var/tmp/filecoin-proof-parameters:/var/tmp/filecoin-proof-parameters \
  --gpus all \
  --hostname fil-2k-miner-miner \
  --name fil-2k-miner-miner \
  -p 1237:2345 \
  lotus-miner:v1.23.0-dev \
  run
```

16. 查看新节点
```
docker exec -it fil-2k-miner-miner lotus-miner info
```

17. 质押扇区（增加节点算力）
```
docker exec -it fil-2k-miner-miner lotus-miner sectors pledge
```

18. 查看扇区ID，获取扇区列表
```
docker exec -it fil-2k-miner-miner lotus-miner sectors list
```

19. 查看扇区信息
```
# 查看具体的某个扇区的状态（示例中显示的是扇区 0 的状态）
docker exec -it fil-2k-miner-miner lotus-miner sectors status 0
# 查看 0 号扇区的详细日志信息
docker exec -it fil-2k-miner-miner lotus-miner sectors status --log 0
```

20. 提交batch
```
# SubmitPreCommitBatch 状态
docker exec -it fil-2k-miner-miner lotus-miner sectors batching precommit --publish-now=true
#  SubmitCommitAggregate 状态
docker exec -it fil-2k-miner-miner lotus-miner sectors batching commit --publish-now=true
```

参考资料  
[Developer network | Filecoin Docs](https://docs.filecoin.io/build/local-devnet/)  
[FileCoin Louts本地2K开发/测试环境部署指南](https://ansheng.me/post/filecoin-louts-local-2k-development-test-environment-deployment-guide)
