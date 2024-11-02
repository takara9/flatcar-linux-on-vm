# Flatcar Linux を QEME/Libvirtで起動する方法
Ubuntu Vitrual Managerがセットアップされている環境で、Flatcar Linux を起動する方法です。


## ストレージの作成

```
sudo -s
export HOSTNAME=flatcar3
export DISK=/var/lib/libvirt/images/boot-${HOSTNAME}.qcow2
export IPADDR_MASK=172.16.0.113/16

modprobe nbd max_part=8
qemu-img create -f qcow2 ${DISK} 20G
qemu-nbd --connect=/dev/nbd0 ${DISK}
fdisk /dev/nbd0 -l
```

## Flatcar　Linuxのインストール
Flatcar Linuxで動かすプロセスなどを増やすには、cl-temp.yamlに記述を追加していきます。
`flatcar-install`は、https://raw.githubusercontent.com/flatcar/init/flatcar-master/bin/flatcar-install から入手できます。
これは、https://www.flatcar.org/docs/latest/installing/bare-metal/installing-to-disk/ からリンクされています。

```
cat cl-temp.yaml|envsubst | docker run --rm -i quay.io/coreos/butane:latest > ignition.json
./flatcar-install -d /dev/nbd0 -i ignition.json
```

## 仮想ディスクの切り離し

```
qemu-nbd --disconnect /dev/nbd0
rmmod nbd
```

## 仮想マシンの起動

 Linuxのパソコンには、2nd NICがセットされており、ブリッジ ネットワーク bridge-host-pri がセットアップされている必要があります。

```
virt-install \
--name ${HOSTNAME} \
--ram=4096 \
--vcpus=1 \
--cpu host \
--hvm \
--disk path=${DISK} \
--graphics none \
--console pty,target_type=serial \
--os-variant linux2022 \
--network network:bridge-host-pri \
--check path_in_use=off \
--boot hd \
--noautoconsole
```

## コンソール接続

```
virsh console ${HOSTNAME} 
```


## VMの削除

```
virsh undefine ${HOSTNAME}
```

