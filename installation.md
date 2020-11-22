# 部署 CoreDNS

## 安装

* 安装

```bash
COREDNS_VERSION="1.3.0"

rm -rf /tmp/coredns* && mkdir -p /tmp/coredns
wget -O /tmp/coredns.tar.gz https://github.com/coredns/coredns/releases/download/v${COREDNS_VERSION}/coredns_${COREDNS_VERSION}_linux_amd64.tgz
tar -zxf /tmp/coredns.tar.gz -C /tmp/coredns
mv /tmp/coredns/coredns /usr/local/bin/ && chmod a+x /usr/local/bin/coredns
rm -rf /tmp/coredns*
```

* 运行

```bash
$ vi /etc/systemd/system/coredns.service
[Unit]
DESCRIPTION=CoreDNS
Documentation=https://coredns.io/manual/toc/
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/local/bin/coredns
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable coredns
systemctl start coredns

systemctl status coredns
journalctl -f -u coredns
```

## Docker

```bash
docker run -d --name=coredns --net=host -p 53:53 coredns/coredns
```

## 参考

* [CoreDNS Installation](https://coredns.io/manual/toc/#installation)
