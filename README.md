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
	
    * shared storage에 연결하여 vm이 공유할 volume을 구성한다. 
	```bash
	systemctl status crio
	rpm -qi cri-o
	```
    * 구성 된 volume에 설치완료된 local storage의 데이터를 이동시킨다.	
    
* 비고 :
    * 추후 설치예정인 network plugin과 crio의 가상 인터페이스 충돌을 막기위해 cri-o의 default 인터페이스 설정을 제거한다.
	```bash
	rm -rf  /etc/cni/net.d/100-crio-bridge
 	rm -rf  /etc/cni/net.d/200-loopback
	``` 
    * 폐쇄망 환경에서 private registry 접근을 위해 crio.conf 내용을 수정한다.
    * insecure_registry, registries, plugin_dirs 내용을 수정한다.
      * vi /etc/crio/crio.conf
         * registries = ["{registry}:{port}" , "docker.io"]
         * insecure_registries = ["{registry}:{port}"]
         * plugin_dirs : "/opt/cni/bin" 추가
	 ![image](figure/crio_config.PNG)
    * crio 사용 전 환경 설정
	```bash
	modprobe overlay
	modprobe br_netfilter
	
	cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
	net.bridge.bridge-nf-call-iptables  = 1
	net.ipv4.ip_forward                 = 1
	net.bridge.bridge-nf-call-ip6tables = 1
	EOF
	```	
    * cri-o를 재시작 한다.
	```bash
	systemctl restart crio
	``` 	
## Step 2. kubeadm, kubelet, kubectl 설치
* 목적 : `Kubernetes 구성을 위한 kubeadm, kubelet, kubectl 설치한다.`
* 순서:
    * CRI-O 메이저와 마이너 버전은 쿠버네티스 메이저와 마이너 버전이 일치해야 한다.
    * kubeadm, kubectl, kubelet 설치 (v1.17.6)
	```bash
	yum install -y kubeadm-1.17.6-0 kubelet-1.17.6-0 kubectl-1.17.6-0
	```  	

## Step 3. kubernetes cluster 구성
* 목적 : `kubernetes master를 구축한다.`
* 순서 :
    * 쿠버네티스 설치시 필요한 kubeadm-config를 작성한다.
        * vi kubeadm-config.yaml
	```bash
	apiVersion: kubeadm.k8s.io/v1beta2
	kind: InitConfiguration
	localAPIEndpoint:
  		advertiseAddress: {master IP}
  		bindPort: 6443
	nodeRegistration:
  		criSocket: /var/run/crio/crio.sock
	---
	apiVersion: kubeadm.k8s.io/v1beta2
	kind: ClusterConfiguration
	kubernetesVersion: v1.17.6
	controlPlaneEndpoint: {master IP}:6443
	imageRepository: “{registry}/k8s.gcr.io”
	networking:
 		serviceSubnet: 10.96.0.0/16
  		podSubnet: "10.244.0.0/16"
	---
	apiVersion: kubelet.config.k8s.io/v1beta1
	kind: KubeletConfiguration
	cgroupDriver: systemd
	```
      * kubernetesVersion : kubernetes version
      * advertiseAddress : API server IP (master IP)
      * serviceSubnet : "${SERVICE_IP_POOL}/${CIDR}"
      * podSubnet : "${POD_IP_POOL}/${CIDR}"
      * imageRepository : "${registry} / docker hub name"
      * cgroupDriver: systemd cgroup driver 사용
	
    * kubeadm init
	```bash
	kubeadm init --config=kubeadm-config.yaml
	```
	![image](figure/init.PNG)
    * kubernetes config 
	```bash
	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config
	```
	![image](figure/success.PNG)
    * 확인
	```bash
	kubectl get nodes
	```
	![image](figure/nodes.PNG)
	```bash
	kubectl get pods -A -o wide
	```
	![image](figure/pods.PNG)	
* 비고 : 
    * single node cluster 구성시 master taint를 제거한다
	```bash
	kubectl taint node [master hostname] node-role.kubernetes.io/master:NoSchedule-
	ex) kubectl taint node k8s- node-role.kubernetes.io/master:NoSchedule- 
	```    

## Step 3-1. kubernetes cluster 다중화 구성을 위한 Keepalived 설치
* 목적 : `K8S cluster의 Master 다중화 구성을 위한 Keepalived를 설치 및 설정한다`
* 순서 : 
    * Keepalived 설치
    ```bash
    yum install -y keepalived
    ```

    * Keepalived 설정
    ```bash
	vi /etc/keepalived/keepalived.conf
	
	vrrp_instance VI_1 {    
	state MASTER    
	interface enp0s8    
	virtual_router_id 50    
	priority 100    
	advert_int 1    
	nopreempt    
	authentication {        
		auth_type PASS        
		auth_pass $ place secure password here.   
		}   
	virtual_ipaddress {        
		VIP  
		} 
	}
    ```
	
	* interface : network interface 이름 확인
	* priority : Master 우선순위
	    * priority 값이 높으면 최우선적으로 Master 역할 수행
	    * 각 Master마다 다른 priority 값으로 수정.
	* virtual_ipaddress : VIP를 입력. Master IP 아님!
	
    * keepalived 재시작 및 상태 확인
    ```bash
    systemctl restart keepalived
    systemctl status keepalived
    ```
	
    * network interface 확인
    ```bash
    ip a
    ```
	
	* 설정한 VIP 확인 가능, 여러 마스터 중 하나만 보임.
	* inet {VIP}/32 scope global eno1

## Step 3-2. kubernetes cluster 다중화 구성 설정
* 목적 : `K8S cluster의 Master 다중화 구성을 위함`
* 순서 : 
    * kubeadm-config.yaml 파일로 kubeadm 명령어 실행.
        * Master 다중구성시 --upload-certs 옵션은 반드시 필요.
	    ```bash
	    kubeadm init --config=kubeadm-config.yaml --upload-certs
	    kubeadm join {IP}:{PORT} --token ~~ --control-plane --certificate-key ~~   (1)
	    kubeadm join {IP}:{PORT} --token ~~   (2)
	    ```
	* 해당 옵션은 certificates를 control-plane으로 upload하는 옵션
	* 해당 옵션을 설정하지 않을 경우, 모든 Master 노드에서 key를 복사해야 함
	* Master 단일구성과는 다르게, --control-plane --certificate-key 옵션이 추가된 명령어가 출력됨
	* (1)처럼 Master 다중구성을 위한 hash 값을 포함한 kubeadm join 명령어가 출력되므로 해당 명령어를 복사하여 다중구성에 포함시킬 다른 Master에서 실행
	* (2)처럼 Worker의 join을 위한 명령어도 출력되므로 Worker 노드 join시 사용
	
	* kubernetes config 
	    ```bash
	    mkdir -p $HOME/.kube
	    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	    sudo chown $(id -u):$(id -g) $HOME/.kube/config
	    ```
