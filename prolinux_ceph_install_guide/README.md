#prolinux 8.2 기준 ceph v15.2.8 설치 간단 가이드

## 필수 패키지
- 다음 패키지는 ceph 설치에 있어서 반드시 필요하므로 ceph를 설치할 모든 노드에서 package 존재 유무 확인 및 설치해주시길 바랍니다. 없으면 설치 중 에러 발생함
	- podman or docker
	- chrony or ntp (없는 경우 있으니 반드시 확인 필요!)
		- 또한, ceph 설치할 노드 간에는 시간 동기화가 반드시 필요하니 chrony 또는 ntp 설치하시고, 노드 간 시간동기화 설정까지 모두 반드시 해주셔야합니다.
	- lvm2

## ceph 설치

1. cephadm image 다운로드 (ceph octopus 최신 branch에 최근까지 버그가 있었으나 이제 해결되었기 때문에 공식 docs url에서 가져오면 됨)
	```shell
	curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
	chmod +x cephadm
	```
2. ceph-common 설치
	- 원래 cephadm을 통해서 설치가 가능하나, cephadm은 prolinux를 지원하지 않기 때문에 내부 os 체크 코드에서 prolinux의 경우 에러가 발생하여 다른 방법으로 설치
		- ./cephadm install, ./cephadm add-repo, ./cephadm rm-repo 사용 불가!!!!!!(에러 발생함)
	- 방법
		1) /etc/yum.repos.d 에 다음 내용의 ceph.repo 파일 추가
		```
		# ceph.repo 파일 

			[Ceph]
			name=Ceph $basearch
			baseurl=https://download.ceph.com/rpm-15.2.8/el8/$basearch
			enabled=1
			gpgcheck=1
			gpgkey=https://download.ceph.com/keys/release.asc

			[Ceph-noarch]
			name=Ceph noarch
			baseurl=https://download.ceph.com/rpm-15.2.8/el8/noarch
			enabled=1
			gpgcheck=1
			gpgkey=https://download.ceph.com/keys/release.asc

			[Ceph-source]
			name=Ceph SRPMS
			baseurl=https://download.ceph.com/rpm-15.2.8/el8/SRPMS
			enabled=1
			gpgcheck=1
			gpgkey=https://download.ceph.com/keys/release.asc
		```	
		2) 다음 커맨드 수행
			```shell
				yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
				yum install ceph-common
			```
3. ceph 설치
	- 설치 전 ceph를 설치할 노드들에 hostname 설정이 되어 있어야합니다. 제대로 되어 있지 않으면 설치 실패함
	```shell
		sudo hostname localhost #host 이름 localhost로 설정
	```
	- bootstrap 명령어를 이용한 ceph 설치 (만약 위의 조건들을 제대로 지키지 않았다면, 여기서 설치가 실패합니다)
		```shell
			# ./cephadm --image {ceph container image 주소} bootstrap --mon-ip {ip주소}
			# octopus 최신 버전을 깔지 않고 특정 버전을 까실거면 반드시 image field를 통해 해당 버전의 container image 주소를 정확히 명시해줘야 합니다
			
			./cephadm --image docker.io/ceph/ceph:v15.2.8 bootstrap --mon-ip 192.168.6.175
		```
	- 여기까지 완료하면 cephadm이 깔린 노드 하나에 ceph를 설치한 것
	- 설치 확인
		```shell
			ceph orch ps
			ceph orch ls
			ceph -s
		```
	- 다른 노드를 ceph cluster에 추가
		- 다음 방법을 통해 노드를 ceph cluster에 추가해야 해당 노드에 ceph daemon들을 추후 설치할 수 있습니다.
		- 참고로 모니터는 5개, mgr은 2 or 3개?가 기본 설정이라 노드를 추가하면 자동으로 해당 노드들에 ceph daemon(mon, mgr)을 배포합니다.
		- host 추가를 위해선 다음 작업을 통해 추가되는 노드에 ssh key 복사 필요
			```shell
				# ssh-copy-id -f -i /etc/ceph/ceph.pub root@*<new-host>*
				
				ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.72.100
			```
		- 커맨드
			```shell
				# ceph orch host add <hostname> <IP>
				
				ceph orch host add node1 192.168.72.100
			```
		- host 추가 확인
			```shell
				ceph orch host ls
			```
4. osd 추가
	- osd 추가는 disk 전체 사용을 전제로 하며, disk는 완벽하게 초기화(partition table, partition, lvm 있으면 안됨)가 되어 있어야 합니다. 
		- disk 초기화
			```shell
				# disk는 /dev/sdb 가정
				
				sgdisk --zap-all /dev/sdb
				dd if=/dev/zero of=/dev/sdb bs=1M count=100 oflag=direct,dsync
				blkdiscard /dev/sdb
				
				# 이전에 ceph를 깐 적이 있는 노드라면 다음 커맨드 수행도 필요
				
				ls /dev/mapper/ceph-* | xargs -I% -- dmsetup remove %
				rm -rf /dev/ceph-*
			```
		- disk ceph 추가 가능여부 확인
			```shell
				# 해당 명령어 쳤을 때 초기화한 disk가 보여야됨
				
				ceph orch device ls --refresh
			```	
	- 노드 단위로 osd 추가
		- 노드 별로 osd 배포를 위한 yaml 파일 생성
		```yaml
		# osd_localhost.yaml 파일
		service_type: osd		#osd로 고정
		service_id: osd_localhost	#마음대로 설정, ex) osd_{hostname}
		placement:
		  hosts:
		  - localhost	#osd 배포할 hostname 명시
		data_devices:
		  paths:
		  - /dev/sdb	#osd 배포할 device 명시
		  - /dev/sdc	#osd 배포할 device 명시
		```	  
		- yaml apply를 통해 osd 배포
			```shell
				ceph orch apply osd -i osd_localhost.yaml
			```	
		- 확인
			```shell
				#osd 추가되었는지 확인
				
				ceph -s 
				ceph osd status
			```
			
5. ceph pool 생성 및 replica size 1로 설정
	- 원래 replication size는 2나 3이 기본이지만, 최소환경 가정으로 1로 설정
	- 참고로, replica 1로 하면 ceph -s시 HEALTH_WARN 발생하는데, ceph cluster는 정상작동합니다
		```shell
			# replicapool : rbd pool
			# myfs-metadata, myfs-data0 : cephfs pool
			# device_health_metrics pool은 ceph cluster 생성시 존재, replica 1로 변경
			
			ceph osd pool set device_health_metrics size 1
			
			# rbd에 사용할 replicapool pool 생성 및 replica 1로 설정
			
			ceph osd pool create replicapool 32 32 replicated
			ceph osd pool set replicapool size 1
			rbd pool init replicapool
			
			# cephfs에 사용할 myfs-metadata, myfs-data0 pool 생성 및 replica 1로 설정까지
			
			ceph osd pool create myfs-metadata 32 32 replicated
			ceph osd pool create myfs-data0 8 8 replicated
			ceph osd pool set myfs-metadata size 1
			ceph osd pool set myfs-data0 size 1
		```
6. cephfs mds 데몬 배포

	```shell
		# ceph orch apply mds {volumename: filesysetem name} --placement="1 {hostname}"
		# 해당 방법 이외에 ceph fs volume create {volumename} {placement}을 통해서도 배포할 수는 있음
		
		ceph orch apply mds myfs --placement="1 localhost"
	```
7. ceph fs 생성
	```shell
		# ceph fs new {volumename: filesysetem name} {medatadata pool} {data pool}
		
		ceph fs new myfs myfs-metadata myfs-data0
	```
8. 확인
	```shell
		ceph -s
		ceph fs status
	```
########
```
cephfs name : myfs
cephfs pool :
	data pool : myfs-data0
	metadata pool : myfs-metadata
rbd pool : replicapool
```

	
	

	