# Flatcar Linux を QEME/Libvirtで起動する方法
Ubuntu Vitrual Managerがセットアップされている環境で、Flatcar Linux を起動する方法です。


## ストレージの作成

```
sudo -s
export HOSTNAME=flatcar1
export DISK=/var/lib/libvirt/images/boot-${HOSTNAME}.qcow2
export IPADDR_MASK=172.16.0.111/16

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


## virsh ブリッジネットワークの設定

```
# virsh net-list
 Name              State    Autostart   Persistent
----------------------------------------------------
 bridge-host-pri   active   yes         yes
 bridge-host-pub   active   yes         yes
 default           active   yes         yes

# virsh net-dumpxml  bridge-host-pri 
<network connections='1'>
  <name>bridge-host-pri</name>
  <uuid>72fdd30c-827e-4352-94ee-083201635e70</uuid>
  <forward mode='bridge'/>
  <bridge name='br1'/>
</network>

# virsh net-dumpxml  bridge-host-pub
<network>
  <name>bridge-host-pub</name>
  <uuid>7de311ee-7f85-420d-9f5c-b6fe01be303d</uuid>
  <forward mode='bridge'/>
  <bridge name='br0'/>
</network>
```

## VMホストのブリッジ設定

マザーボードに、２ポートのNICを追加してあります。
NICは、10Gtek 10/100/1000Mbps ギガビット イーサネット PCI Express NIC ネットワークカード,Intel 82571コントローラー, デュアル RJ45ポートで Amazonから入手できます。
購入時の金額 3,999円 

```
# brctl show
bridge name     bridge id               STP enabled     interfaces
br-901072bb2d9e         8000.02429bd70e41       no
br0             8000.ca606fecc010       yes             enp3s0f0
br1             8000.2e741c84430b       yes             enp3s0f1
```

## 問題解決

- https://bbs.archlinux.org/viewtopic.php?id=197950