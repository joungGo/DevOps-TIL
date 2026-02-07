# 📘 Vagrantfile 이해를 위한 Ruby 기본 문법 학습 가이드

> 목적
> 
> - Vagrantfile을 **설정 파일이 아닌 Ruby 코드로 이해**
>     
> - Ruby DSL(Vagrant)의 구조를 읽고 해석할 수 있는 수준 도달
>     

---

## 1️⃣ Ruby 파일의 정체

### ✔ Ruby는 인터프리터 언어

- `.rb` 파일은 **위에서 아래로 순차 실행**
    
- Vagrantfile도 **그냥 Ruby 코드**
    

```ruby
cfg.vm.box = "generic/ubuntu2004"
```

→ 설정이 아니라 **메서드 호출**

---

## 2️⃣ 주석 (Comment)

```ruby
# 한 줄 주석
```

- `#` 뒤는 실행되지 않음
    
- 여러 줄 주석 문법은 없음 (모두 `#`)
    

```ruby
# config.vm.define "node01" do |cfg|
#   ...
# end
```

---

## 3️⃣ 상수 (Constant)

```ruby
Vagrant_API_Version = "2"
```

### 문법 규칙

- **대문자로 시작** → 상수
    
- 변경은 가능하지만 **관례적으로 변경하지 않음**
    

### 용도

- 설정 값, 버전 정보 등
    

---

## 4️⃣ 메서드 호출 기본 형태

### 기본형

```ruby
method_name(arg1, arg2)
```

### Ruby 특징: 괄호 생략 가능

```ruby
method_name arg1, arg2
```

📌 Vagrantfile에서 자주 등장

```ruby
cfg.vm.synced_folder ".", "/vagrant"
```

---

## 5️⃣ 블록(Block) 문법 ⭐⭐⭐ (핵심)

### 기본 구조

```ruby
method do |variable|
  실행 코드
end
```

### 실제 예제

```ruby
Vagrant.configure("2") do |config|
  ...
end
```

### 의미

- `configure` 메서드가
    
- **객체 하나를 블록 변수(`config`)로 전달**
    

👉 블록은 **콜백 + 설정 컨텍스트**

---

## 6️⃣ 블록 중첩 구조

```ruby
config.vm.define "k8s-master" do |cfg|
  cfg.vm.provider :virtualbox do |vb|
    vb.cpus = 2
  end
end
```

### 해석 순서

1. `define` → VM 하나 정의
    
2. `provider` → 가상화 플랫폼 설정
    
3. `vb` → provider 전용 객체
    

---

## 7️⃣ 객체와 메서드 호출

```ruby
cfg.vm.box = "generic/ubuntu2004"
```

### 실제 Ruby 내부 동작

```ruby
cfg.vm.box=("generic/ubuntu2004")
```

✔ `=` 은 **대입이 아니라 setter 메서드 호출**

---

## 8️⃣ 심볼(Symbol)

### 문법

```ruby
:virtualbox
:"k8s-master"
```

### 특징

|항목|문자열|심볼|
|---|---|---|
|변경 가능|O|X|
|메모리|큼|작음|
|용도|데이터|키/식별자|

### Vagrant에서 사용 예

```ruby
cfg.vm.provider :virtualbox
```

---

## 9️⃣ 해시(Hash) & 키워드 인자

### 해시 기본 문법

```ruby
{ key: "value" }
```

### 메서드 인자로 사용

```ruby
cfg.vm.network "private_network", ip: "192.168.32.10"
```

→ 내부적으로

```ruby
cfg.vm.network("private_network", { ip: "192.168.32.10" })
```

---

## 🔟 멀티 인자 메서드 호출

```ruby
cfg.vm.network "forwarded_port",
  guest: 22,
  host: 21022,
  auto_correct: false
```

✔ 줄 바꿈 가능  
✔ 마지막 쉼표 허용

---

## 1️⃣1️⃣ 문자열 (String)

```ruby
"generic/ubuntu2004"
```

- 큰따옴표 / 작은따옴표 동일
    
- Vagrantfile에서는 큰따옴표가 관례
    

---

## 1️⃣2️⃣ 변수

```ruby
cfg
vb
config
```

### 특징

- 타입 선언 없음
    
- 객체 참조
    

👉 전부 **Vagrant 내부 객체**

---

## 1️⃣3️⃣ DSL (Domain Specific Language)

Vagrantfile 문법이 쉬워 보이는 이유:

```ruby
cfg.vm.network "private_network", ip: "192.168.32.10"
```

실제로는:

```ruby
cfg.vm.network(
  "private_network",
  { ip: "192.168.32.10" }
)
```

✔ Ruby 문법 + 사람이 읽기 쉬운 DSL 설계

---

## 1️⃣4️⃣ 전체 문법 구조 요약

```
Ruby
 ├─ 상수
 ├─ 메서드 호출
 ├─ 블록
 ├─ 객체
 ├─ 심볼
 ├─ 해시
 └─ DSL
```

---

## 1️⃣5️⃣ 이 정도면 가능한 것

이 가이드 수준이면:

✅ Vagrantfile 읽기  
✅ 네트워크/포트 설정 이해  
✅ 멀티 VM 구조 파악  
✅ 문법 에러 구분 가능

❌ Ruby 앱 개발 (그건 다음 단계)

---

## 🔚 다음 학습 추천

원하면 이어서:

1. 📦 [[클래스 개념 (최소한만)]]
    
2. 🧠 [[Vagrant DSL 내부 동작 원리]]
    
3. 🌐 [[forwarded_port → NAT → iptables 흐름]]
    
4. 🛠 [[Vagrantfile 실습 과제 (주석 해제 & 확장)]]
    