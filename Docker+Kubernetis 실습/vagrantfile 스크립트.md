```ruby
# -*- mode: ruby -*-                     # 이 파일이 ruby 문법임을 에디터에 알림
# vi: set ft=ruby :                      # vi/vim에서 ruby 파일 타입으로 인식하도록 설정

Vagrant_API_Version="2"                  # Vagrant 설정 파일(API) 버전 지정

Vagrant.configure(Vagrant_API_Version) do |config|  # Vagrant 설정 시작
  # Node01
  config.vm.define "k8s-node01" do |cfg|  # k8s-node01 이라는 VM 정의
    cfg.vm.box = "generic/ubuntu2004"     # 사용할 베이스 박스(OS 이미지)
    cfg.vm.box_version = "3.1.16"         # 박스의 버전 고정 (환경 동일성 유지)
    cfg.vm.provider :virtualbox do |vb|  # VirtualBox provider 설정
      vb.name = "K8s-Node01"              # VirtualBox에 표시될 VM 이름
      vb.cpus = 1                         # CPU 코어 수
      vb.memory = 1024                    # 메모리(MB 단위)
    end
    cfg.vm.hostname = "k8s-node01"        # VM 내부에서 사용할 호스트명
    cfg.vm.network "private_network", ip: "192.168.32.11"  # Host-only 네트워크 IP 설정
    cfg.vm.network "forwarded_port", guest: 22, host: 24022, auto_correct: false, id: "ssh"
                                             # 게스트(22번 SSH 포트) → 호스트 24022 포트로 포워딩
    cfg.vm.network "forwarded_port", guest: 80, host: 40080
                                             # 웹 서버(80) → 호스트 40080
    cfg.vm.network "forwarded_port", guest: 8080, host: 48080
                                             # 애플리케이션 포트(8080) → 호스트 48080
    cfg.vm.provision "shell", path: "sshd_config.sh"
                                             # VM 생성 시 sshd 설정을 위한 쉘 스크립트 실행
  end

  # # Node02 (현재 주석 처리되어 비활성화됨)
  # config.vm.define "k8s-node02" do |cfg| # 두 번째 워커 노드 정의
  #   cfg.vm.box = "generic/ubuntu2004"    # Ubuntu 20.04 박스 사용
  #   cfg.vm.provider :virtualbox do |vb|
  #       vb.name = "K8s-Node02"            # VirtualBox VM 이름
  #       vb.cpus = 1                       # CPU 1개
  #       vb.memory = 1024                  # 메모리 1GB
  #   end
  #   cfg.vm.hostname = "k8s-node02"        # VM 호스트명
  #   cfg.vm.network "private_network", ip: "192.168.32.12" # 내부 통신용 IP
  #   cfg.vm.network "forwarded_port", guest: 22, host: 23022, auto_correct: false, id: "ssh"
  #                                            # SSH 포트 포워딩
  #   cfg.vm.network "forwarded_port", guest: 80, host: 30080
  #                                            # 웹 포트 포워딩
  #   cfg.vm.network "forwarded_port", guest: 8080, host: 38080
  #                                            # 애플리케이션 포트 포워딩
  #   cfg.vm.provision "shell", path: "sshd_config.sh"
  #                                            # SSH 설정 스크립트 실행
  # end

  # Master
  config.vm.define "k8s-master" do |cfg|  # Kubernetes Master 노드 정의
    cfg.vm.box = "generic/ubuntu2004"     # Ubuntu 20.04 사용
    cfg.vm.provider :virtualbox do |vb|
        vb.name = "K8s-Master"             # VirtualBox에 표시될 이름
        vb.cpus = 2                        # Master 노드이므로 CPU 2개
        vb.memory = 2048                   # 메모리 2GB
    end
    cfg.vm.hostname = "k8s-master"         # Master 노드 호스트명
    cfg.vm.synced_folder ".", "/vagrant"   # 호스트 현재 디렉토리를 VM의 /vagrant에 마운트
    cfg.vm.network "private_network", ip: "192.168.32.10"
                                             # 클러스터 내부 통신용 IP
    cfg.vm.network "forwarded_port", guest: 22, host: 21022, auto_correct: false, id: "ssh"
                                             # SSH 접속용 포트 포워딩
    cfg.vm.network "forwarded_port", guest: 80, host: 10080
                                             # 웹 서비스 포트
    cfg.vm.network "forwarded_port", guest: 8000, host: 18000
                                             # 서비스 테스트용 포트
    cfg.vm.network "forwarded_port", guest: 8001, host: 18001
                                             # 추가 서비스 포트
    cfg.vm.network "forwarded_port", guest: 8080, host: 18080
                                             # 애플리케이션 포트
    cfg.vm.provision "shell", path: "sshd_config.sh"
                                             # SSH 설정 스크립트 실행
  end
end
```