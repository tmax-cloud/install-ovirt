# ovirt-install-guide

# ovirt hosted-engine 설치 가이드

## 구성 요소 및 버전
* Prolinux 8
* oVirt 4.4.3
* ceph-common.x86_64 2:15.2.5-1.el8

## 환경구성
```
+-----------------------+          |          +-----------------------+
|   [   Admin Node   ]  | 10.0.0.1 | 10.0.0.5 | [    oVirt Engine   ] |
|    ovirt1.tmax.dom    +----------+----------+     master.tmax.dom   |
|                       |          |          |                       |
+-----------------------+          |          +-----------------------+
                                   |
+-----------------------+          |
| [   Shared Storage  ] |10.0.0.6  |
|     ceph.tmax.dom     +----------+
|                       |
+-----------------------+
```
## Prerequisites

1. oVirt 패키지와 ceph 패키지 설치가 가능한 repository를 모든 노드에 추가합니다.
    * 티맥스 타워의 서버의 경우
        * /etc/yum.repo.d/Ovirt.repo 생성
        ```bash 
        [OvirtRepo]
        name=ovirt-repo
        baseurl=http://pldev-repo-21.tk/prolinux/ovirt/4.4/el8/x86_64/
        gpgcheck=0 

        [CephRepo]
        name=ceph-repo
        baseurl=http://vault.centos.org/8.2.2004/storage/x86_64/ceph-octopus
        gpgcheck=0
        ```
    * Private 환경의 경우 private repository 주소로 수정합니다.
    * 아래의 명령어를 통해 repository가 정상적으로 추가되었는지 확인합니다.
    ```bash
    $ yum repolist
    ```
    
2. 각 노드와 engine VM에 대한 domain을 정의합니다.
    * Ovirt Cluster에 대한 private DNS가 있는 경우 해당 DNS에 각 노드에 대한 domain name을 추가합니다.
    * private DNS가 없는 경우 모든 노드의 /etc/hosts에 각 노드와 engine VM에 대한 domain name을 추가합니다.
        * /etc/hosts 예시
        ```bash
        10.0.0.2 ovirt1.tmax.dom ovirt1   # admin node
        10.0.0.3 ovirt2.tmax.dom ovirt2   # another node
        10.0.0.5 master.tmax.dom master   # engine VM
        10.0.1.2 ceph.tmax.dom   ceph     # storage node
        ```  
    
3.  cephfs를 engine VM의 ovirt storage domain으로 사용하기 위해 cephfs volume을 준비합니다.
    * 구성된 ceph cluster로부터 uid와 gid가 36이고 크기가 300GB인 cephfs volume을 생성합니다.
    * 생성된 volume에 대한 path와 접근하기 위한 user name, user secret 정보를 가져옵니다.
        * CEPHFS_PATH = mount하고자 하는 cephfs에 대한 경로
        * MOUNT_OPTION = 연결하고자 하는 ceph의 secret 정보
        * 예시
        ```bash
        CEPHFS_PATH - 10.0.1.2:6789:/volumes/_nogroup/vol/abcdefgh-1111-2222-3333-abcdefghijkl
        MOUNT_OPTION - name=admin,secret=ABCdEfGftAEvExAAultsKBpNpiWWGi06Md7kks==
        ```

## Install Steps
0. [패키지 설치](https://github.com/tmax-cloud/ovirt-install-guide/tree/master/K8S_Master#step0-%ED%99%98%EA%B2%BD-%EC%84%A4%EC%A0%95)
1. [oVirt engine 설치](https://github.com/tmax-cloud/ovirt-install-guide/tree/master/K8S_Master#step-1-cri-o-%EC%84%A4%EC%B9%98)
2. [oVirt ha 구성](https://github.com/tmax-cloud/ovirt-install-guide/tree/master/K8S_Master#step-2-kubeadm-kubelet-kubectl-%EC%84%A4%EC%B9%98)


## Step0. 패키지 설치
* 목적 : `hosted-engine, ceph설치를 위한 패키지 설치`
* 타겟 : admin node, ha로 구성될 another node들 
* 순서 :
    * module 설정
    ```bash
    $ sudo dnf module disable virt
    $ sudo dnf module enable pki-deps postgresql:12 parfait
    ```
    * hosted-engine 패키지 설치
    ```bash
    $ yum install -y ovirt-hosted-engine-setup
    ```
    * ceph-common 설치
    ```bash
    $ yum install -y ceph-common
    ```
    * script 배포
    ```bash
    $ tar -xvf ovirt4.4.3_posixfs_scripts.tar && cd ovirt4.4.3_posixfs_scripts
    $ ./deploy.sh
    ```

## Step 1. oVirt engine 설치
* 목적 : `oVirt domain을 제어할 engine을 vm위에 배포한다.`
* 실행 : 
    ```bash
    $ hosted-engine --deploy
    ```
* 순서 :
    * local storage에 vm을 기동하여 engine을 설치한다. 
    
    ```markdown
    Are you sure you want to continue? (Yes, No)[Yes]:`Enter`

    Please indicate the gateway IP address [172.21.7.1]:`Enter`

    Please indicate a nic to set ovirtmgmt bridge on: (eno1) [eno1]:`Enter` (다른 node와 통신 가능한 interface)

    Please specify which way the network connectivity should be checked (ping, dns, tcp, none) [dns]:`Enter`

    Please enter the name of the datacenter where you want to deploy this hosted-engine host. [Default]:`Enter`
    
    Please enter the name of the cluster where you want to deploy this hosted-engine host. [Default]:`Enter`

    If you want to deploy with a custom engine appliance image,
    please specify the path to the OVA archive you would like to use
    (leave it empty to skip, the setup will use ovirt-engine-appliance rpm installing it if missing):
    Please specify the number of virtual CPUs for the VM (Defaults to appliance OVF value): [4]: `${ENGINE_NODE_CPU_SIZE}`

    Please specify the memory size of the VM in MB (Defaults to appliance OVF value): [16384]: `${ENGINE_NODE_MEMORY_SIZE}`
    
    Detecting host timezone.
    Please provide the FQDN you would like to use for the engine.
    Note: This will be the FQDN of the engine VM you are now going to launch,
    it should not point to the base host or to any other existing machine.
    Engine VM FQDN:  []: `${ENGINE_NODE_DNS}` (Prerequisites 2에 적은 engine VM의 domain name)
    
    Please provide the domain name you would like to use for the engine appliance.
    Engine VM domain: [tmax.dom] `Enter`
    
    Enter root password that will be used for the engine appliance: `${ENGINE_NODE_ROOT_PASSWD}`
        
    Confirm appliance root password:  `${ENGINE_NODE_ROOT_PASSWD_CONFIRM}`
    
    Enter ssh public key for the root user that will be used for the engine appliance (leave it empty to skip): `Enter`
        
    Skipping appliance root ssh public key
    Do you want to enable ssh access for the root user (yes, no, without-password) [yes]: `Enter`

    Do you want to apply a default OpenSCAP security profile (Yes, No) [No]: `Enter`
    
    You may specify a unicast MAC address for the VM or accept a randomly generated default [00:16:3e:20:25:8f]: `Enter`
    
    How should the engine VM network be configured (DHCP, Static)[DHCP]? `static`

    Please enter the IP address to be used for the engine VM []: `${ENGINE_HOST_STATIC_IP}` (Prerequisites 2에 적은 engine VM의 IP)
    
    Engine VM DNS (leave it empty to skip) [8.8.8.8]: `Enter`
    
    Add lines for the appliance itself and for this host to /etc/hosts on the engine VM?
    Note: ensuring that this host could resolve the engine VM hostname is still up to you
    (Yes, No)[No]  `yes`

    Please provide the name of the SMTP server through which we will send notifications [localhost]: `Enter`

    Please provide the TCP port number of the SMTP server [25]:`Enter`

    Please provide the email address from which notifications will be sent [root@localhost]: `Enter`

    Please provide a comma-separated list of email addresses which will get notifications [root@localhost]: `Enter`

    Enter engine admin password: `${ENGINE_ADMIN_PAGE_PASSWD}`
    Confirm engine admin password: `${ENGINE_ADMIN_PAGE_PASSWD_CONFIRM}`

    Stage: Setup validation
    Please provide the hostname of this host on the management network [ovirt1.tmax.dom]: `Enter`
    ```
    
    * shared storage에 연결정보를 설정 및 연결 한다.    
    ```markdown
    Please provide the hostname of this host on the management network [ovirt1.tmax.dom]: `Enter`
    
    Please specify the storage you would like to use (glusterfs, iscsi, fc, nfs, posixfs)[nfs]: `posixfs`
    
    Please specify the vfs type you would like to use (ext4, ceph, nfs)[ceph]: `ceph`
    
    Please specify the full shared storage connection path to use (example: host:/path): `${CEPHFS_PATH}`

    If needed, specify additional mount options for the connection to the hosted-engine storagedomain (example: rsize=32768,wsize=32768) []:  `${MOUNT_OPTION}`
    ```
    
    * 공유할 volume을 구성하고 설치완료된 local storage의 데이터를 이동시킨다.
    ```markdown
    Please specify the size of the VM disk in GiB: [101]: 120
    ```
* 확인 
    * Admin node에서 engine vm 상태 확인
    ```bash
    $ Hosted-engine --vm-status 
    ```
    * ovirt-engine page 확인 
        * url: https://${ENGINE_NODE_DNS}/ovirt-engine
        * ex) https://master.tmax.dom/ovirt-engine
    
## Step 2. oVirt ha 구성
* 목적 : `engine의 ha구성을 위한 node를 추가한다.`
* 구성 : 
    ```
    +-----------------------+                     +-----------------------+
    |   [   Admin Node   ]  | 10.0.0.1 | 10.0.0.5 | [    oVirt Engine   ] |
    |    ovirt1.tmax.dom    +----------+----------+     master.tmax.dom   |
    |                       |          |          |                       |
    +-----------------------+          |          +-----------------------+
                                       |
    +-----------------------+          |          +-----------------------+
    | [   Shared Storage  ] | 10.0.0.6 | 10.0.0.2 |  [   oVirt Node     ] |
    |     ceph.tmax.dom     +----------+----------+    ovirt2.tmax.dom    |
    |                       |                     |                       |
    +-----------------------+                     +-----------------------+

    ```
* 순서: 
    * libvirtd 및 cockpit.socket 활성화   
    ```bash
    $ systemctl enable --now libvirtd cockpit.socket
    ```
    
    * https://${ENGINE_NODE_DNS}/ovirt-engine에 접속하여 oVirt Admin Portal에서 id는 admin, 비밀번호는 설치때 설정한 ${ENGINE_ADMIN_PAGE_PASSWD}로 로그인합니다.
    * 왼쪽 대시보드에서 컴퓨팅[Compute] - 호스트[Hosts]를 클릭합니다.
    * 새로만들기[New]를 클릭합니다.
    * 추가하고자 하는 노드의 정보를 입력합니다. Required items are Name/HostName of target Node and authentication method
        * 일반[General]탭에서 Name에는 적절한 host 이름을 적고, Hostname/IP에는 domain name을 입력하고 password를 설정합니다.
        * 호스트 엔진[Hosted Engine]탭에서 없음을 배포로 바꾸고 아래의 확인[OK]를 누릅니다.
    * 상태[Status]가 Installing으로 바뀌는 것을 확인합니다. 성공적으로 노드 추가가 되면, 상태[Status]는 Up으로 바뀝니다.

* 확인:
    * Admin Node 혹은 추가된 node에서 hosted-engine --vm-status를 통해 추가된 node에 vm이 down된 형태로 추가되었는지 확인 
