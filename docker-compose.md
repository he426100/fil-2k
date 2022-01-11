1. 准备
```
# 首次
docker volume create lotus-data-repo
docker run --rm -v `pwd`/exported:/src -v lotus-data-repo:/dest busybox /bin/sh -c "cp /src/tmp/localnet.json /dest/ && cp -r /src/tmp/.genesis-sectors /dest/ && chown 532:532 -R /dest"

docker volume create lotus-parameters
# 已有证明参数
docker run --rm -v /var/tmp/filecoin-proof-parameters:/src -v lotus-parameters:/dest busybox /bin/sh -c "cp /src/* /dest/ && chown 532:532 -R /dest"
```

2. 运行
```
docker-compose up
```
