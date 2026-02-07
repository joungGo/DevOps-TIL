# 2️⃣ Vagrant DSL 내부 동작 원리

## 2-1️⃣ DSL이란?

**DSL (Domain Specific Language)**  
→ 특정 목적을 위해 만든 “읽기 쉬운 언어”

Vagrantfile은:

- Ruby 문법
    
- - 인프라 설정 전용 DSL
        

---

## 2-2️⃣ 왜 이렇게 생겼을까?

`cfg.vm.network "private_network", ip: "192.168.32.10"`

사람이 읽으면:

> “이 VM에 private 네트워크를 추가하고 IP는 이거야”

기계가 읽으면:

`cfg.vm.network(   "private_network",   { ip: "192.168.32.10" } )`

✔ Ruby의 **메서드 + 해시 + 블록**을 이용한 설계

---

## 2-3️⃣ define 블록의 진짜 의미

`config.vm.define "k8s-master" do |cfg|   ... end`

내부적으로:

1. VM 하나 생성
    
2. VM 설정 객체 생성
    
3. 그 객체를 `cfg`로 넘김
    
4. 블록 안의 설정을 누적
    

👉 **실행이 아니라 “설정 수집”**

---

## 2-4️⃣ provider 블록의 의미

`cfg.vm.provider :virtualbox do |vb|   vb.cpus = 2 end`

✔ “VirtualBox일 경우에만 적용되는 설정”  
✔ provider가 바뀌면 이 블록은 무시될 수 있음