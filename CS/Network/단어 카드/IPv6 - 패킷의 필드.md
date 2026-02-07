## IPv6 패킷 주요 필드 정리

|필드명|위치|역할|비고|
|---|---|---|---|
|다음 헤더 (Next Header)|IPv6 기본 헤더|다음에 오는 헤더 종류 지정|확장 헤더 또는 L4 프로토콜|
|식별자 (Identification)|Fragment 확장 헤더|단편화된 패킷 식별|기본 헤더에는 없음|
|단편화 오프셋 (Fragment Offset)|Fragment 확장 헤더|조각 위치 정보|8바이트 단위|
|M 플래그 (More Fragments)|Fragment 확장 헤더|추가 단편 존재 여부|1: 있음 / 0: 마지막|
|홉 제한 (Hop Limit)|IPv6 기본 헤더|최대 홉 수 제한|IPv4 TTL과 동일|
|송신 IP 주소|IPv6 기본 헤더|출발지 주소|128비트|
|수신 IP 주소|IPv6 기본 헤더|목적지 주소|128비트|

---

## 한 줄 요약

IPv6는 기본 헤더를 단순화하고, 단편화 관련 정보는 Fragment 확장 헤더로 분리하여 처리한다.

---

# IPv6 패킷의 필드 설명

## 1. 다음 헤더 (Next Header)

### 의미

- IPv6에서 **현재 헤더 다음에 오는 헤더의 종류**를 나타내는 필드
    
- IPv4의 `Protocol` 필드 역할을 확장한 개념
    

### 역할

- 다음에 오는 것이
    
    - 확장 헤더인지
        
    - TCP/UDP/ICMPv6 같은 상위 계층인지 구분
        

### 예시 값

- 0 : Hop-by-Hop Options
    
- 44 : Fragment Header
    
- 6 : TCP
    
- 17 : UDP
    
- 58 : ICMPv6
    

### 핵심 포인트

- IPv6는 **고정 헤더 + 확장 헤더 체인 구조**
    
- Next Header를 따라가며 순차 해석
    

---

## 2. 식별자 (Identification)

### 위치

- **IPv6 기본 헤더에는 없음**
    
- **Fragment 확장 헤더**에만 존재
    

### 의미

- 단편화된 패킷 조각들을 구분하기 위한 식별 값
    

### 특징

- **송신 호스트만 단편화 수행**
    
- 라우터는 단편화하지 않음
    

---

## 3. 단편화 오프셋 (Fragment Offset)

### 위치

- Fragment 확장 헤더
    

### 의미

- 해당 조각의 **원본 데이터 내 위치**
    
- 단위: 8바이트
    

### 역할

- 수신 측에서 조각 재조립
    

---

## 4. M 플래그 (More Fragments)

### 위치

- Fragment 확장 헤더
    

### 의미

- 뒤에 **추가 단편이 있는지 여부**
    

|값|의미|
|---|---|
|1|뒤에 더 있음|
|0|마지막 조각|

### IPv4와 차이

- IPv4: Flags 필드 안에 MF 존재
    
- IPv6: Fragment 헤더로 분리
    

---

## 5. 홉 제한 (Hop Limit)

### 의미

- 패킷이 통과할 수 있는 **최대 홉 수**
    

### IPv4와 대응

- IPv4의 TTL과 동일한 역할
    

### 동작 방식

- 라우터 통과 시마다 1 감소
    
- 0이 되면 패킷 폐기 + ICMPv6 Time Exceeded
    

---

## 6. 송수신 IP 주소 (Source / Destination Address)

### 길이

- 각각 **128비트**
    

### 역할

- 송신 IP: 출발지 식별
    
- 수신 IP: 최종 목적지 판단
    

### 특징

- 라우팅 판단 기준
    
- 기본적으로 중간 라우터에서 변경되지 않음
    

---

## IPv6 단편화 구조 요약

```text
IPv6 기본 헤더
 ├─ Next Header → Fragment Header
       ├─ Identification
       ├─ Fragment Offset
       └─ M Flag
 └─ Next Header → TCP / UDP / ICMPv6
```

---

## IPv4 vs IPv6 단편화 비교 요약

|항목|IPv4|IPv6|
|---|---|---|
|단편화 위치|IP 기본 헤더|Fragment 확장 헤더|
|라우터 단편화|가능|불가|
|단편화 주체|송신/라우터|송신만|
|헤더 구조|가변|고정 + 확장|

---

## 한 문장 요약

IPv6는 기본 헤더를 단순화하고, 단편화와 상위 계층 정보는 확장 헤더와 Next Header 체인을 통해 처리함으로써 라우팅 성능과 구조적 명확성을 높였다.