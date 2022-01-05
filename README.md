1. 基于Dockerfile.lotus构建镜像
```
git clone https://github.com/filecoin-project/lotus.git && cd lotus && git checkout v1.13.1
docker build -t lotus:v1.13.1-dev --target lotus --build-arg -f Dockerfile.lotus-debug .
docker build -t lotus-seed:v1.13.1-dev --target lotus-seed --build-arg -f Dockerfile.lotus-debug .
docker build -t lotus-miner:v1.13.1-dev --target lotus-miner --build-arg -f Dockerfile.lotus-debug .
docker build -t lotus-worker:v1.13.1-dev --target lotus-worker --build-arg -f Dockerfile.lotus-debug .
```
官方的Dockerfile.lotus和scripts/docker-lotus-miner-entrypoint.sh有问题

2. 预密封两个2KiB大小的扇区
```
mkdir -p /tmp/fil-2k-data && sudo chown 532:532 /tmp/fil-2k-data

docker run --rm \
  -v /tmp/fil-2k-data:/data \
  he426100/lotus-seed:v1.13.1-dev \
  --sector-dir /data/.genesis-sectors \
  pre-seal --sector-size=2KiB --num-sectors=2
```

3. 创建创世块
```
docker run --rm \
  -v /tmp/fil-2k-data:/data \
  he426100/lotus-seed:v1.13.1-dev \
  genesis new /data/localnet.json
```

4. 使用一些FIL为默认帐户提供资金
```
docker run --rm \
  -v /tmp/fil-2k-data:/data \
  he426100/lotus-seed:v1.13.1-dev \
  --sector-dir /data/.genesis-sectors \
  genesis add-miner /data/localnet.json /data/.genesis-sectors/pre-seal-t01000.json
```

5. 启动第一个节点
```
mkdir -p /tmp/fil-2k-lotus && sudo chown 532:532 /tmp/fil-2k-lotus

docker run -d -it \
  -e LOTUS_API_LISTENADDRESS="/ip4/0.0.0.0/tcp/1234/http" \
  -e DOCKER_LOTUS_IMPORT_SNAPSHOT="" \
  -v /tmp/fil-2k-data:/data \
  -v /tmp/fil-2k-lotus:/var/lib/lotus \
  -v /var/tmp/filecoin-proof-parameters:/var/tmp/filecoin-proof-parameters \
  --hostname fil-2k-master-lotus \
  --name fil-2k-master-lotus \
  -p 1234:1234 \
  he426100/lotus:v1.13.1-dev \
  daemon --lotus-make-genesis=/data/devgen.car --genesis-template=/data/localnet.json --bootstrap=false
```

6. 导入创世矿工密钥
```
docker exec -it fil-2k-master-lotus lotus wallet import --as-default /data/.genesis-sectors/pre-seal-t01000.key
docker exec -it fil-2k-master-lotus lotus wallet list
# 查看token
FULLNODE_API_INFO=$(docker exec -it fil-2k-master-lotus lotus auth api-info --perm admin | awk -F '=' '{print $2}')
LOTUS_IP=$(docker exec -it fil-2k-master-lotus cat /etc/hosts | grep fil-2k-master-lotus | awk '{print $1}')
FULLNODE_API_INFO=${FULLNODE_API_INFO/0.0.0.0/$LOTUS_IP}
echo $FULLNODE_API_INFO
```

7. 初始化并运行创世矿工
```
mkdir -p /tmp/fil-2k-lotus-miner && sudo chown 532:532 /tmp/fil-2k-lotus-miner
# 不知道为啥不能用2345端口
docker run -d -it \
  -e LOTUS_API_LISTENADDRESS="/ip4/0.0.0.0/tcp/2345/http" \
  -e FULLNODE_API_INFO="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBbGxvdyI6WyJyZWFkIiwid3JpdGUiLCJzaWduIiwiYWRtaW4iXX0.gC7e-eH921Ag7363ArTvVzfF75HC6cH3HhefkmmKiWY:/ip4/172.17.0.4/tcp/1234/http" \
  -e DOCKER_LOTUS_MINER_INIT=true \
  -e DOCKER_LOTUS_MINER_INIT_ARGS="--genesis-miner --actor=t01000 --sector-size=2KiB --pre-sealed-sectors=/data/.genesis-sectors --pre-sealed-metadata=/data/.genesis-sectors/pre-seal-t01000.json --nosync" \
  -v /tmp/fil-2k-data:/data \
  -v /tmp/fil-2k-lotus-miner:/var/lib/lotus-miner \
  -v /var/tmp/filecoin-proof-parameters:/var/tmp/filecoin-proof-parameters \
  --hostname fil-2k-master-miner \
  --name fil-2k-master-miner \
  -p 1235:2345 \
  he426100/lotus-miner:v1.13.1-dev \
  run --nosync
```
