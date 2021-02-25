# ovirt-install-guide

# ovirt hosted-engine 설치 가이드

## 구성 요소 및 버전
* Prolinux 8
* oVirt 4.4
* ceph-common.x86_64 2:15.2.5-1.el8

## 환경구성
```
+-----------------------+          |          +-----------------------+
|   [   Admin Node   ]  | 10.0.0.1 | 10.0.0.5 | [    oVirt Engine   ] |
|    ovirt1.test.dom    +----------+----------+     master.test.dom   |
|                       |          |          |                       |
+-----------------------+          |          +-----------------------+
                                   |
+-----------------------+          |
| [   Shared Storage  ] |10.0.0.6  |
|     ceph.test.dom     +----------+
|                       |
+-----------------------+
```
## Prerequisites

1. oVirt 패키지와 ceph 패키지 다운로드를 위해 oVirt repository를 등록한다
    * repository 설정
        * /etc/yum.repo.d/Ovirt.repo 생성
    ```bash 
    [OvirtRepo]
    name=ovirt-repo
    baseurl=http://pldev-repo-21.tk/prolinux/ovirt/4.4/el8/x86_64/
    gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-prolinux-$releasever-release
    gpgcheck=0
    
    [CephRepo]
    name=ceph-repo
    baseurl=http://vault.centos.org/8.2.2004/storage/x86_64/ceph-octopus
    gpgcheck=0
    ```
    * repository 등록 확인
    ``` bash
    $ yum repolist
    ```    
    
2.  domain 구성을 위해 admin node와 engine node의 dns및 hostname을 등록한다.
    * /etc/hosts파일에 추가
        * DNS는 HOSTNAME과 DOMAIN을 합친 형태로 구성        
        * ex) HOSTNAME=ovirt1 이면 DNS는 ovirt1.test.dom으로 구성
    * 일반 format
    ```bash    
    ${ADMIN_NODE_IP} ${ADMIN_NODE_DNS} ${ADMIN_NODE_HOSTNAME}
    ${ENGINE_NODE_IP} ${ENGINE_NODE_DNS} ${ENGINE_NODE_HOSTNAME}
    ```
    * 예제
    ```bash
    172.21.7.9 ovirt1.test.dom ovirt1    # admin node
    172.21.7.10 ovirt2.test.dom ovirt2   # another node
    172.21.7.11 ovirt3.test.dom ovirt3   # another node
    172.21.7.17 master.test.dom master   # engine node on VM
    ```  
    
3.  cephs filesystem를 shared storage로 사용하기 위해 shared storage에서 ceph filesystem을 요청한다. 
    


## Install Steps
0. [패키지 설치](https://github.com/tmax-cloud/ovirt-install-guide/tree/master/K8S_Master#step0-%ED%99%98%EA%B2%BD-%EC%84%A4%EC%A0%95)
1. [oVirt engine 설치](https://github.com/tmax-cloud/ovirt-install-guide/tree/master/K8S_Master#step-1-cri-o-%EC%84%A4%EC%B9%98)
2. [oVirt ha 구성](https://github.com/tmax-cloud/ovirt-install-guide/tree/master/K8S_Master#step-2-kubeadm-kubelet-kubectl-%EC%84%A4%EC%B9%98)


## Step0. 패키지 설치
* 목적 : `hosted-engine, ceph설치를 위한 패키지 설치`
* 순서 : 
    * hosted-engine 패키지 설치
	```bash
	$ yum install -y ovirt-hosted-engine-setup
	```
    * ceph-common 설치
	```bash
	$ yum install -y ceph-common
	```  

## Step 1. oVirt engine 설치
* 목적 : `oVirt domain을 제어할 engine을 vm위에 배포한다.`
* 실행 : 
    ```bash
    $ hosted-engine --deploy
    ```
* 순서 :
    * local storage에 vm을 기동하여 engine을 설치한다. 
        ```bash	
	Are you sure you want to continue? (Yes, No)[Yes]:`Enter`

  	Please indicate the gateway IP address [172.21.7.1]:`Enter`

 	Please indicate a nic to set ovirtmgmt bridge on: (eno1) [eno1]:`Enter`

	Please specify which way the network connectivity should be checked (ping, dns, tcp, none) [dns]:

	Please enter the name of the datacenter where you want to deploy this hosted-engine host. [Default]:

	Please enter the name of the cluster where you want to deploy this hosted-engine host. [Default]:

	If you want to deploy with a custom engine appliance image,
	please specify the path to the OVA archive you would like to use
	(leave it empty to skip, the setup will use ovirt-engine-appliance rpm installing it if missing):
	Please specify the number of virtual CPUs for the VM (Defaults to appliance OVF value): [4]: ${ENGINE_HOST_VM_CPU_SIZE}

	Please specify the memory size of the VM in MB (Defaults to appliance OVF value): [16384]: ${ENGINE_HOST_VM_MEMORY_SIZE}

	Detecting host timezone.
	Please provide the FQDN you would like to use for the engine.
	Note: This will be the FQDN of the engine VM you are now going to launch,
	it should not point to the base host or to any other existing machine.
	Engine VM FQDN:  []: ${ENGINE_DNS} ->

	Please provide the domain name you would like to use for the engine appliance.
	Engine VM domain: [test.dom]

	Enter root password that will be used for the engine appliance: ${ENGINE_HOST_ROOT_PASSWD}

	Confirm appliance root password:  ${ENGINE_HOST_ROOT_PASSWD}

	Enter ssh public key for the root user that will be used for the engine appliance (leave it empty to skip):

	Skipping appliance root ssh public key
	Do you want to enable ssh access for the root user (yes, no, without-password) [yes]:

	Do you want to apply a default OpenSCAP security profile (Yes, No) [No]:

	You may specify a unicast MAC address for the VM or accept a randomly generated default [00:16:3e:20:25:8f]:

	How should the engine VM network be configured (DHCP, Static)[DHCP]? static

	Please enter the IP address to be used for the engine VM []: ${ENGINE_HOST_STATIC_IP}

	Engine VM DNS (leave it empty to skip) [8.8.8.8]:

	Add lines for the appliance itself and for this host to /etc/hosts on the engine VM?
	Note: ensuring that this host could resolve the engine VM hostname is still up to you
	(Yes, No)[No] yes

	Please provide the name of the SMTP server through which we will send notifications [localhost]:

	Please provide the TCP port number of the SMTP server [25]:

	Please provide the email address from which notifications will be sent [root@localhost]:

	Please provide a comma-separated list of email addresses which will get notifications [root@localhost]:

	Enter engine admin password:
	Confirm engine admin password:

	Stage: Setup validation
	Please provide the hostname of this host on the management network [ovirt1.test.dom]:
	```
    * shared storage에 연결하여 vm이 공유할 volume을 구성한다. 
	```bash
	systemctl status crio
	rpm -qi cri-o
	```
    * 구성 된 volume에 설치완료된 local storage의 데이터를 이동시킨다.	
    
* 비고 :

