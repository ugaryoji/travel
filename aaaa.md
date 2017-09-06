# インストール手順【準備作業)
## ansibleのインストール
### 作業対象サーバ
- [x] AFMのインストールサーバ
- [ ] app-platform
- [ ] compute
- [ ] openstack-controller
### 作業内容
* ansibleをインストールします。
環境変数にproxyの設定が必要になりますので、インストール前に下記の設定を追加してください。
```
export http_proxy="http://itsdev01:8208314114@rep.proxy.nic.fujitsu.com:8080/"
export https_proxy="http://itsdev01:8208314114@rep.proxy.nic.fujitsu.com:8080/"
```
* inventory配下を設定します。  
hostsを下記のように設定。IPやansible_ssh_user、ansible_ssh_private_key_fileは適時修正してください。
```
[appformix_controller]
172.29.1.40
[compute]
172.29.1.41
[openstack_controller]
172.29.1.42
[appformix_controller:vars]
ansible_ssh_port=22
ansible_ssh_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/monitoring-dev.pem
[compute:vars]
ansible_ssh_port=22
ansible_ssh_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/monitoring-dev.pem
[openstack_controller:vars]
ansible_ssh_port=22
ansible_ssh_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/monitoring-dev.pem
```
group_vars/all.ymlを以下のように設定します。パスとバージョンは適時修正してください。
```
appformix_license: /home/ubuntu/AppFormix2.9/appformix-license-APPFXT00000008.sig
appformix_docker_images: ["/home/ubuntu/AppFormix2.9/appformix-platform-images-2.9.1.tar.gz","/home/ubuntu/AppFormix2.9/appformix-openstack-images-2.9.1.tar.gz"]
```
* ansibleにプロキシ設定を追加  
プロキシ設定を追加します。`roles/proxy/vars/main.yml`ファイルに以下の設定を追加します。
no_proxyが無いと、内部のローカルIPの接続先に連携できなくなります。
インストール実行中に接続処理があるので設定してください。
```
https_proxy: "http://itsdev01:8208314114@rep.proxy.nic.fujitsu.com:8080/"
http_proxy: "http://itsdev01:8208314114@rep.proxy.nic.fujitsu.com:8080/"
no_proxy: "172.29.1.40,172.29.1.41,172.29.1.42"
```

## プロキシの設定
### 作業対象サーバ
- [ ] AFMのインストールサーバ
- [x] app-platform
- [x] compute
- [x] openstack-controller
### 作業内容
`/etc/environment`に以下の設定を追加してください。  
注：インストール作業後は削除した方が良いかも。要確認
```
export http_proxy="http://itsdev01:8208314114@rep.proxy.nic.fujitsu.com:8080/"
export https_proxy="http://itsdev01:8208314114@rep.proxy.nic.fujitsu.com:8080/"
export NO_PROXY=localhost,172.29.1.40,172.29.1.41,172.29.1.42
```
NO_PROXYで設定したIPはAFMをインストールすサーバのIP  

## /etc/hostsに各サーバ情報を設定
### 作業対象サーバ
- [ ] AFMのインストールサーバ
- [x] app-platform
- [x] compute
- [x] openstack-controller
### 作業内容
`/etc/hosts`に各サーバの情報を下記のように設定  
```
172.29.1.40 afm-app-platform01
172.29.1.41 afm-compute01
172.29.1.42 afm-openstack-controller01
```

## pythonのインストール
### 作業対象サーバ
- [x] AFMのインストールサーバ
- [x] app-platform
- [x] compute
- [x] openstack-controller
### 作業内容
pythonと関連ライブラリをインストールします。手順は下記の通り
```
sudo -E apt-get -y update
sudo -E apt-get -y upgrade
sudo -E apt-get -y install build-essential
sudo -E apt-get -y install libsqlite3-dev
sudo -E apt-get -y install libreadline6-dev
sudo -E apt-get -y install libgdbm-dev
sudo -E apt-get -y install zlib1g-dev
sudo -E apt-get -y install libbz2-dev
sudo -E apt-get -y install sqlite3
sudo -E apt-get -y install tk-dev
sudo -E apt-get -y install zip
sudo -E apt-get -y install libssl-dev
sudo -E apt-get -y install python-dev
sudo -E wget https://bootstrap.pypa.io/get-pip.py
sudo -E python get-pip.py
```

## dockerのインストール
### 作業対象サーバ
- [x] AFMのインストールサーバ
- [x] app-platform
- [ ] compute
- [ ] openstack-controller
### 作業内容
dockerと関連ライブラリをインストールします。手順は下記の通り
```
source env.file
sudo -E apt-get update
sudo -E apt-get install apt-transport-https ca-certificates
echo "deb https://apt.dockerproject.org/repo ubuntu-xenial main" | sudo tee /etc/apt/sources.list.d/docker.list
sudo -E apt-get update
sudo -E apt-get install linux-image-extra-$(uname -r) linux-image-extra-virtual
sudo -E apt-get update
sudo -E apt-get install docker-engine
```
dockerにproxyの設定を行います。`/lib/systemd/system/docker.service`を修正します。  
下記の文言を[Service]配下に追加します。
`Environment='http_proxy=http://itsdev01:8208314114@rep.proxy.nic.fujitsu.com:8080/'`  
修正後、下記のコマンドを実行し、設定を読み込ませて下さい。
```
sudo systemctl daemon-reload
sudo systemctl restart docker
```
下記のコマンドでdockerの起動でubuntuに権限を与えてください。
```
sudo groupadd docker
sudo usermod -g docker ubuntu
sudo /bin/systemctl restart docker.service
```

## ip6tablesの動作確認
### 作業対象サーバ
- [ ] AFMのインストールサーバ
- [x] app-platform
- [x] compute
- [x] openstack-controller
### 作業内容
ip6tablesの動作確認を行ってください。
OKな場合
```
ubuntu@ansible-server-test-uga:~$ sudo ip6tables -L
sudo: unable to resolve host ansible-server-test-uga
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination  
```
NGな場合。カーネルをアップグレードする必要があると出ます。カーネルをアップグレードしてもダメな場合は、
インストールするubuntuのイメージを見直してください。
```
ubuntu@appformix-platform:~$ sudo ip6tables -L
ip6tables v1.6.0: can't initialize ip6tables table `filter': Address family not supported by protocol
Perhaps ip6tables or your kernel needs to be upgraded.
ubuntu@appformix-platform:~$ ip6tables -L
ip6tables v1.6.0: can't initialize ip6tables table `filter': Address family not supported by protocol
Perhaps ip6tables or your kernel needs to be upgraded.
```

# インストール時に発生する異常について色々
### TASK [docker : Prerequisites for using HTTPS apt repository] 
```
TASK [docker : Prerequisites for using HTTPS apt repository] *********************************************************************************************************************
failed: [172.29.1.4] (item=[u'apt-transport-https', u'ca-certificates', u'software-properties-common']) => {"cmd": "apt-get update", "failed": true, "item": ["apt-transport-https", "ca-certificates", "software-properties-common"], "msg": "W: The repository 'http://security.ubuntu.com/ubuntu xenial-security Release' does not have a Release file.\nW: The repository 'http://archive.ubuntu.com/ubuntu xenial Release' does not have a Release file.\nW: The repository 'http://archive.ubuntu.com/ubuntu xenial-updates Release' does not have a Release file.\nW: The repository 'http://archive.ubuntu.com/ubuntu xenial-backports Release' does not have a Release file.\nW: Failed to fetch https://apt.dockerproject.org/repo/dists/ubuntu-xenial/InRelease  Could not resolve host: apt.dockerproject.org\nE: Failed to fetch http://security.ubuntu.com/ubuntu/dists/xenial-security/main/source/Sources  Something wicked happened resolving 'security.ubuntu.com:http' (-5 - No address associated with hostname)\nE: Failed to fetch http://archive.ubuntu.com/ubuntu/dists/xenial-backports/restricted/source/Sources  Something wicked happened resolving 'archive.ubuntu.com:http' (-5 - No address associated with hostname)\nE: Failed to fetch http://archive.ubuntu.com/ubuntu/dists/xenial/main/source/Sources  Something wicked happened resolving 'archive.ubuntu.com:http' (-5 - No address associated with hostname)\nE: Failed to fetch http://archive.ubuntu.com/ubuntu/dists/xenial-updates/main/source/Sources  Something wicked happened resolving 'archive.ubuntu.com:http' (-5 - No address associated with hostname)\nW: Some index files failed to download. They have been ignored, or old ones used instead.", "rc": 100, "stderr": "W: The repository 'http://security.ubuntu.com/ubuntu xenial-security Release' does not have a Release file.\nW: The repository 'http://archive.ubuntu.com/ubuntu xenial Release' does not have a Release file.\nW: The repository 'http://archive.ubuntu.com/ubuntu xenial-updates Release' does not have a Release file.\nW: The repository 'http://archive.ubuntu.com/ubuntu xenial-backports Release' does not have a Release file.\nW: Failed to fetch https://apt.dockerproject.org/repo/dists/ubuntu-xenial/InRelease  Could not resolve host: apt.dockerproject.org\nE: Failed to fetch http://security.ubuntu.com/ubuntu/dists/xenial-security/main/sou
```
### 解決方法
ネットワーク異常が発生しています。プロキシの設定を全て行っているか確認してください。

### docker権限無し異常:TASK [appformix_docker_images : Load AppFormix Docker images from file]
```
TASK [docker : Install docker-py] ************************************************************************************************************************************************
fatal: [172.29.1.4]: FAILED! => {"changed": false, "cmd": "/usr/local/bin/pip2 install docker-py==1.3.1", "failed": true, "msg": "stdout: Collecting docker-py==1.3.1\n\n:stderr:   Retrying (Retry(total=4, connect=None, read=None, redirect=None)) after connection broken by 'NewConnectionError('<pip._vendor.requests.packages.urllib3.connection.VerifiedHTTPSConnection object at 0x7fc2ced6fd50>: Failed to establish a new connection: [Errno -5] No address associated with hostname',)': /simple/docker-py/\n  Retrying (Retry(total=3, connect=None, read=None, redirect=None)) after connection broken by 'NewConnectionError('<pip._vendor.requests.packages.urllib3.connection.VerifiedHTTPSConnection object at 0x7fc2ced6f050>: Failed to establish a new connection: [Errno -5] No address associated with hostname',)': /simple/docker-py/\n  Retrying (Retry(total=2, connect=None, read=None, redirect=None)) after connection bro

```

### 解決方法
dockerの起動でubuntuに権限を与える
```
sudo groupadd docker					
sudo usermod -g docker ubuntu					
sudo /bin/systemctl restart docker.service					
```