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
    
3.  cephfs를 ovirt storage domain으로 사용하기 위한 설정 
    * ceph 로부터 생성된 cephfs 연결 PATH및 mount option 정보를 기반으로 한다.
    * admin node에서 임의의 directory에 mount한다.
        ```bash
	$ mount -t ceph ${CEPHFS_PATH} ${TEMP_DIR} -o ${MOUNT_OPTION}
	```
    * 임의 directory에 권한을 부여한다. 
        ```bash
	$ chmod 777 ${TEMP_DIR}
        ```
## Install Steps
0. [패키지 설치](https://github.com/tmax-cloud/ovirt-install-guide/tree/master/K8S_Master#step0-%ED%99%98%EA%B2%BD-%EC%84%A4%EC%A0%95)
1. [oVirt engine 설치](https://github.com/tmax-cloud/ovirt-install-guide/tree/master/K8S_Master#step-1-cri-o-%EC%84%A4%EC%B9%98)
2. [oVirt ha 구성](https://github.com/tmax-cloud/ovirt-install-guide/tree/master/K8S_Master#step-2-kubeadm-kubelet-kubectl-%EC%84%A4%EC%B9%98)


## Step0. 패키지 설치
* 목적 : `hosted-engine, ceph설치를 위한 패키지 설치`
* 타겟 : admin node, ha로 구성될 another node들 
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
    
        ```markdown	
		Are you sure you want to continue? (Yes, No)[Yes]:`Enter`

  		Please indicate the gateway IP address [172.21.7.1]:`Enter`

 		Please indicate a nic to set ovirtmgmt bridge on: (eno1) [eno1]:`Enter`

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
		Engine VM FQDN:  []: `${ENGINE_NODE_DNS}`
	
		Please provide the domain name you would like to use for the engine appliance.
		Engine VM domain: [test.dom] `Enter`
	
		Enter root password that will be used for the engine appliance: `${ENGINE_NODE_ROOT_PASSWD}`
		
		Confirm appliance root password:  `${ENGINE_NODE_ROOT_PASSWD_CONFIRM}`
	
		Enter ssh public key for the root user that will be used for the engine appliance (leave it empty to skip): `Enter`
		
		Skipping appliance root ssh public key
		Do you want to enable ssh access for the root user (yes, no, without-password) [yes]: `Enter`

		Do you want to apply a default OpenSCAP security profile (Yes, No) [No]: `Enter`
	
		You may specify a unicast MAC address for the VM or accept a randomly generated default [00:16:3e:20:25:8f]: `Enter`
	
		How should the engine VM network be configured (DHCP, Static)[DHCP]? `static`
	
		Please enter the IP address to be used for the engine VM []: `${ENGINE_HOST_STATIC_IP}` <-- ex)10.0.0.5
	
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
		Please provide the hostname of this host on the management network [ovirt1.test.dom]: `Enter`
        ```
    
    * shared storage에 연결정보를 설정 및 연결 한다.    
	```markdown
		Please provide the hostname of this host on the management network [ovirt1.test.dom]: `Enter`
		
        	Please specify the storage you would like to use (glusterfs, iscsi, fc, nfs, posixfs)[nfs]: `posixfs`
		
        	Please specify the vfs type you would like to use (ext4, ceph, nfs)[ceph]: `ceph`
		
        	Please specify the full shared storage connection path to use (example: host:/path): `${CEPH_MOUNT_PATH}`    <-- ex)172.21.3.8:6789:/volumes/_nogroup/tim3/17a08a7a-1f51-43b0-b399-2bca4bffe5ac
		
          	If needed, specify additional mount options for the connection to the hosted-engine storagedomain (example: rsize=32768,wsize=32768) []:  `${CEPH_MOUNT_OPTION}` <-- ex) name=admin,secret=AQCeBcZftAEvExAAultsKBpNpiWWGi06Md7mmw==		
	```
	
    *  공유할 volume을 구성하고 설치완료된 local storage의 데이터를 이동시킨다.	
        ```markdown
		Please specify the size of the VM disk in GiB: [101]: 120 `${SHARED_DISK_SIZE_FOR_VM}`
	```
* 확인 
    * Admin node에서 engine vm 상태 확인
    ```bash
    	$ Hosted-engine --vm-status 
    ```
    * ovirt-engine page 확인 
        * url: https://${ENGINE_NODE_DNS}/ovirt-engine
        * ex) https://master.test.dom/ovirt-engine
    

