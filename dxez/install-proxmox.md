# Install proxmox

### 가상화 구성을 위한 proxmox 설치

* 일반 linux 설치 절차와 유사하며 자세한 설치는 https://www.linuxtechi.com/install-proxmox-ve-on-bare-metal/ 를 참조한다.
* Table of Contents

```
   Prerequisites
1) Download Proxmox VE ISO Installer
2) Boot from the Proxmox VE Installer
3) Accept End User License Agreement
4) Choose Installation Target Disk
5) Set Location and Time Zone
6) Create Admin Password and Email Address
7) Network Configuration
8) Begin Proxmox VE Installation
9) Access the Proxmox VE Web Interface
10) Change Proxmox Enterprise Repository to Community
```

* proxmox 서버에 TLS 설정 (도메인 이름이 handymes.com 일경우)

```bash
$ git clone  https://github.com/acmesh-official/acme.sh.git acme.sh-master
$ cd acme.sh-master/
$ sudo ./acme.sh --register-account -m zasmin@uengine.net
$ sudo ./acme.sh --issue --standalone --keypath /etc/pve/local/pve-ssl.key --server letsencrypt --fullchainpath /etc/pve/local/pve-ssl.pem --reloadcmd "systemctl restart pveproxy" -d proxmox3.handymes.com
```

* proxmox public licenses 관련 팝업 제거

```bash
$ cd /usr/share/javascript/proxmox-widget-toolkit/
$ cp proxmoxlib.js proxmoxlib.js.bak
$ vi proxmoxlib.js
    Ext.Msg.show({      ----->     void ({ // Ext.Msg.show({
$ sudo systemctl restart pveproxy.service
```
