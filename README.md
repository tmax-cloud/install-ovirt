# ovirt-install-guide

## 구성 요소 및 버전
* Prolinux 8
* oVirt 4.4.3
* ceph-common.x86_64 2:15.2.5-1.el8

## Prerequisites

1. oVirt 패키지와 ceph 패키지 설치가 가능한 repository를 모든 노드에 추가합니다.
    * Public Network 환경의 경우
        * /etc/yum.repo.d/ovirt.repo 생성
        ```bash 
        [OvirtRepo]
        name=ovirt-repo
        baseurl=http://prolinux-repo.tmaxos.com/ovirt/4.4/el8/x86_64/
        gpgcheck=0 
        ```
    * Private Network 환경의 경우
        * /etc/yum.repos.d/ProLinux.repo 의 baseurl을 ProLinux를 위한 private repository 주소로 수정합니다.
        * 위의 OvirtRepo의 baseurl을 Ovirt를 위한 private repository 주소로 수정합니다.
    * 아래의 명령어를 통해 repository가 정상적으로 추가되었는지 확인합니다.
    ```bash
    $ yum repolist
    ```
    
2. 모든 노드와 engine VM에 대한 domain을 정의합니다.
    * oVirt cluster에 대한 private DNS가 있는 경우 해당 DNS에 각 노드에 대한 domain name을 추가합니다.
    * private DNS가 없는 경우 모든 노드의 /etc/hosts에 각 노드와 engine VM에 대한 domain name을 추가합니다.
        * /etc/hosts 예시
        ```bash
        10.0.0.2 engine.tmax.dom master  # for engine VM
        10.0.0.3 node1.tmax.dom node1    # for physical node1
        10.0.0.4 node2.tmax.dom node2    # for physical node2
        ```  
    
3. cephfs를 engine VM의 ovirt storage domain으로 사용하기 위해 cephfs volume을 준비합니다.
    * 구성된 ceph cluster로부터 uid와 gid가 36이고 크기가 300GB인 cephfs volume을 생성합니다.
         ```
         $ ceph fs subvolume create {{ fs name }} {{ subvolume name }} --size 322122547200 --uid 36 --gid 36
         $ ceph fs subvolume getpath {{ fs name }} {{ subvolume name }} => CEPHFS_PATH로 사용
         $ cat /etc/ceph/ceph.client.admin.keyring => MOUNT_OPTION의 secret으로 사용
         ```
    * 생성된 volume에 대한 path와 접근하기 위한 user name, user secret 정보를 가져옵니다.
        * CEPHFS_PATH = mount하고자 하는 cephfs에 대한 경로
        * MOUNT_OPTION = 연결하고자 하는 ceph의 secret 정보
        * 예시
        ```bash
        CEPHFS_PATH - 10.0.1.2:6789:/volumes/_nogroup/vol/abcdefgh-1111-2222-3333-abcdefghijkl
        MOUNT_OPTION - name=admin,secret=ABCdEfGftAEvExAAultsKBpNpiWWGi06Md7kks==
        ```
        
4. Ceph와 동일한 클러스터에 oVirt가 설치되는 경우, 각 노드에서 ceph이 사용하는 local device들을 multipath blacklist에 추가합니다.
    * 아래의 명령어를 통해 device에 대한 wwid를 가져옵니다. (ceph이 사용하는 모든 device들에 대해 수행합니다)
    ```bash
    $ /lib/udev/scsi_id --whitelisted --device=/dev/sdb
    35002538d40a75c92
    ```
    * /etc/multipath/conf.d/ceph.conf 를 열고 blacklist를 작성합니다.
    ```bash
    $ mkdir -p /etc/multipath/conf.d
    $ vi /etc/multipath/conf.d/ceph.conf
    blacklist {
        wwid "35002538d40a75c92"
        wwid "35000039fe9d4449f"
    }
    ```

5. engine VM의 HA를 위해 최소 2대의 물리 노드가 필요합니다.

## Install Steps
0. [패키지 설치](https://github.com/tmax-cloud/ovirt-install-guide/tree/master/K8S_Master#step0-%ED%99%98%EA%B2%BD-%EC%84%A4%EC%A0%95)
1. [oVirt engine 설치](https://github.com/tmax-cloud/ovirt-install-guide/tree/master/K8S_Master#step-1-cri-o-%EC%84%A4%EC%B9%98)
2. [oVirt ha 구성](https://github.com/tmax-cloud/ovirt-install-guide/tree/master/K8S_Master#step-2-kubeadm-kubelet-kubectl-%EC%84%A4%EC%B9%98)


## Step0. 패키지 설치
* 목적: `hosted-engine, ceph설치를 위한 패키지 설치`
* 타겟: oVirt clutser로 구성될 모든 노드 
* 순서:
    * module 설정
    ```bash
    $ sudo dnf module disable virt -y
    $ sudo dnf module enable pki-deps postgresql:12 parfait -y
    ```
    * hosted-engine 패키지 설치
    ```bash
    $ yum install -y ovirt-hosted-engine-setup
    ```
    * ceph-common 설치
    ```bash
    $ yum install -y ceph-common
    ```

## Step 1. oVirt engine 설치
* 목적: `oVirt domain을 제어할 engine을 vm위에 배포`
* 순서:
    * 아래의 명령어를 수행하여 VM에 할당할 mac address를 생성합니다. 이 값을 아래의 변수중 OVEHOSTED_vmMACAddr에 설정합니다.
    ```bash
    $ python3.6 -c "from ovirt_hosted_engine_setup import util as ohostedutil; print(ohostedutil.randomMAC())"
    ```
    * answers.conf 에서 아래의 변수들에 대해 {xxx} 값을 원하는 환경설정의 값으로 수정합니다.
        * OVEHOSTED_ENGINE/adminPassword=str:{UI admin 계정 비밀번호}
        * OVEHOSTED_NETWORK/bridgeIf=str:{다른 물리 노드와 통신 가능한 interface 이름}
        * OVEHOSTED_NETWORK/fqdn=str:{Engine VM에 할당할 domain name}
        * OVEHOSTED_NETWORK/gateway=str:{Engine VM의 network gateway 주소}
        * OVEHOSTED_NETWORK/host_name=str:{현재 deploy를 진행하는 노드의 domain name}
        * OVEHOSTED_STORAGE/mntOptions=str:{cephfs volume에 접근하기 위한 user 정보 (prerequisite의 MOUNT_OPTION)}
        * OVEHOSTED_STORAGE/storageDomainConnection=str:{cephfs volume에 접근하기 위한 주소 (prerequisite의 CEPH_PATH)}
        * OVEHOSTED_VM/cloudinitInstanceDomainName=str:{위의 OVEHOSTED_NETWORK/fqdn 에서 subdomain을 뺀 domain name}
        * OVEHOSTED_VM/cloudinitInstanceHostName=str:{위의 OVEHOSTED_NETWORK/fqdn과 같은 값}
        * OVEHOSTED_VM/cloudinitRootPwd=str:{Engine VM의 root 계정에 대한 password}
        * OVEHOSTED_VM/cloudinitVMDNS=str:{Engine VM이 사용할 dns}
        * OVEHOSTED_VM/cloudinitVMStaticCIDR=str:{Engine VM의 static ip에 대한 CIDR}
        * OVEHOSTED_VM/vmMACAddr=str:{Engine VM의 mac 주소 (위의 명령어를 통해 얻은 값으로 설정)}
        * OVEHOSTED_VM/vmMemSizeMB=int:{EngineVM의 memory 크기 (최소: 4096, 권장: 16384)}
        * OVEHOSTED_VM/vmVCpus=str:{EngineVM의 vcpu 수 (최소: 2, 권장: 4)}
        * OVEHOSTED_VM/proLinuxRepoAddress=str:{prolinux repository 주소 (private network인 경우에만 설정)}
        * OVEHOSTED_VM/ovirtRepoAddress=str:{ovirt repository 주소 (private network인 경우에만 설정)}
    * 아래의 명령어를 통해 deploy를 수행합니다. (20 ~ 30분 소요)
    ```bash
    $ hosted-engine --deploy --config-append=answers.conf
    ```
    * 아래의 명령어를 통해 engine VM의 상태를 확인합니다.\
    Engine status의 "vm"이 "up"이고 "health"가 "good"인 경우 정상적으로 설치가 완료된 것입니다.
    ```bash
    $ hosted-engine --vm-status 
    ```
    * ovirt-engine UI 접속 URL
        * URL: https://{OVEHOSTED_NETWORK/fqdn}/ovirt-engine
        * ex) https://engine.tmax.dom/ovirt-engine
* 예시:
    * cluster 구성
        * 물리 노드 2, 스토리지 노드 1
        * Engine VM에 10.0.0.2 - engine.tmax.dom 할당
    ```
                                  gateway
                                  10.0.0.1
                                      |
    +----------------------+          |
    |  [     Node 1     ]  |          |          +----------------------+
    |    node1.tmax.dom    | 10.0.0.3 | 10.0.0.4 |  [     Node 2     ]  |
    |                 eno1-+----------+----------+    node2.tmax.dom    |
    |                      |          |          |                      |
    +----------------------+          |          +----------------------+
                                      |
    +----------------------+          |
    | [  Shared Storage  ] | 10.0.1.2 |
    |    ceph.tmax.dom     +----------+
    |                      |
    +----------------------+
    ```
    * configuration
        * answers_example.conf를 참조하세요.

## Step 2. oVirt node 추가
* 목적: `oVirt cluster에 새로운 node를 추가`
* 순서: 
    * libvirtd 및 cockpit.socket 활성화   
    ```bash
    $ systemctl enable --now libvirtd cockpit.socket
    ``` 
    * https://{OVEHOSTED_NETWORK/fqdn}/ovirt-engine에 접속하여 oVirt Admin Portal에서 id는 admin, 비밀번호는 설치때 설정한 {OVEHOSTED_ENGINE/adminPassword}로 로그인합니다.
    * 왼쪽 대시보드에서 컴퓨팅[Compute] - 호스트[Hosts]를 클릭합니다.
    * 새로만들기[New]를 클릭합니다.
    * 추가하고자 하는 노드의 정보를 입력합니다.
        * 일반[General]탭에서 Name에는 적절한 host 이름을 적고, Hostname/IP에는 domain name을 입력하고 password를 설정합니다.
        * 해당 노드를 engine을 위한 HA node로 사용하고자할 경우, 호스트 엔진[Hosted Engine]탭에서 없음을 배포로 바꾸고 아래의 확인[OK]를 누릅니다. (최대 7개의 노드까지 HA node로 설정 가능합니다)
    * 상태[Status]가 Installing으로 바뀌는 것을 확인합니다. 성공적으로 노드 추가가 되면, 상태[Status]는 Up으로 바뀝니다.
    * HA node로 설정한 경우 아래의 명령어를 통해 추가된 node에 vm이 down된 형태로 추가되었는지 확인합니다.
    ```bash
    $ hosted-engine --vm-status
    ```
 * 만일 ceph와 동일 클러스터상의 노드를 oVirt노드로 추가하는 경우 다음 작업을 추가로 실시합니다.
   * lvm.conf 파일 내용 수정
   ```
   vi /etc/lvm/lvm.conf

   filter 검색
   # filter = ["a|^/dev/disk/by-id/lvm-pv-uuid-zQgFIQ-oHef-Hyk9-oZwq-RF7r-P0WP-mRSE0A$|", "r|.*|"]
   위와 같은 내용을 찾아서 주석처리.  

   reboot # 호스트 재부팅 실시.
   ```
## MBS 설치 가이드 
   * https://github.com/tmax-cloud/hypersds-wiki/tree/main/ovirt_mbs 참조.
