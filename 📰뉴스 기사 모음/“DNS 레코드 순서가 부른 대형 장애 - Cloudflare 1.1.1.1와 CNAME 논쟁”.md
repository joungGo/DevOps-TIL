Cloudflare 1.1.1.1의 메모리 최적화 업데이트가 DNS 응답에서 CNAME과 A 레코드의 **순서**를 바꾸면서, 일부 DNS 클라이언트에서 대규모 장애가 난 사건을 정리한 글이다.[[news.hada](https://news.hada.io/topic?id=25981)]​

## 사건의 핵심 요약

- 2026년 1월 8일, Cloudflare의 1.1.1.1에 메모리 사용량 절감을 위한 코드가 배포되면서 DNS 응답 내 레코드 순서가 바뀌었고, 전 세계 일부 사용자에게 DNS 해석 실패가 발생했다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
- 기존에는 CNAME 체인이 먼저, 그 뒤에 A/AAAA 레코드가 오던 구조였는데, 코드 변경으로 CNAME이 응답의 마지막으로 밀리면서, 순서에 의존하던 일부 클라이언트 구현이 응답을 “비어 있다”고 보고 실패했다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

## 기술적 원인과 영향

- glibc의 getaddrinfo는 응답을 순차적으로 읽으면서 “현재 이름”을 추적하기 때문에 CNAME이 먼저 나와야 다음 쿼리 이름을 올바르게 따라갈 수 있고, 뒤에 나오면 결과가 빈 값이 된다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
- Cisco Catalyst 스위치 일부 모델의 DNSC 프로세스도 이 순서에 의존하고 있었고, 1.1.1.1을 사용하던 장비에서는 치명적인 오류로 재부팅 루프가 발생해 Cisco가 별도 서비스 문서를 낼 정도의 사고가 되었다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

## 표준(RFC)과 모호성 문제

- RFC 1034는 “응답은 하나 이상의 CNAME으로 **preface(앞에 붙을 수)** 있다”고만 쓰고, MUST/SHOULD 같은 규범적 단어 없이 CNAME과 A 레코드 간의 순서를 명확히 강제하지 않는다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
- 같은 문서에서 “응답 섹션의 RR 순서 차이는 중요하지 않다”는 예시가 있어, Cloudflare는 이를 일반 규칙으로 해석했지만, 글과 댓글에서는 “특정 예시에만 해당하는 문장을 전체 규칙으로 일반화한 잘못된 해석”이라는 비판이 제기된다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

## 구현별 차이와 교훈

- systemd-resolved는 응답을 OrderedSet 형태로 파싱해 레코드 순서와 무관하게 전체 집합을 탐색하기 때문에 이번 변경에도 영향을 받지 않았다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
- 반대로 glibc 같은 단순 스텁 리졸버는 캐시도 없고, 빠른 조회를 위한 복잡한 자료구조도 없어 순서에 덜 의존하도록 고치려면 큰 구조 변경이 필요하다는 지적이 나온다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

## Cloudflare의 대응과 커뮤니티 논평

- Cloudflare는 “RFC가 CNAME 순서를 요구하지 않는다”는 입장을 내세우면서도, 현실적으로 많은 클라이언트가 CNAME이 앞에 온다고 가정하고 있다는 점을 인정하고, 정책을 “CNAME은 항상 먼저 배치”로 되돌렸다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
- 동시에 DNS 응답 내 CNAME 처리 순서에 대한 명확한 표준을 만들기 위해 IETF에 draft-jabley-dnsop-ordered-answer-section이라는 Internet-Draft를 제출했으며, 이 사건이 “40년 된 DNS 프로토콜의 모호성이 여전히 실무를 깨뜨릴 수 있음”을 보여주는 사례로 정리된다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

## 댓글에서 나온 추가 논의 포인트

- “possibly preface”를 어떻게 해석해야 하는지, 그리고 “MUST가 없다고 해서 아무 순서나 괜찮다는 뜻은 아니다”라는 언어·규범 논쟁이 이어진다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
- Hyrum’s Law(사용자가 많아지면 가능한 모든 동작에 누군가는 의존하게 된다)와 Postel’s Law(보내는 것은 보수적으로, 받는 것은 관대하게)가 충돌한 사례로 보는 의견, 그리고 Postel의 법칙이 오늘날에는 오히려 해로운 원칙으로 여겨진다는 비판도 있다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

## DNS 운용·설계 측면의 시사점

- CNAME은 이름 레벨의 별칭이라는 특성 때문에 apex CNAME, CNAME+기타 레코드(예: TXT) 조합 등에서 오래전부터 혼란과 버그를 만들어 왔고, 이번 사건은 그런 “누수된 추상화”가 대규모 인프라에서 한 번에 터진 사례라는 평가가 나온다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
- 댓글에서는 Cloudflare가 과거에도 표준을 어기고(예: apex CNAME 허용, RFC 8482) 나중에 새로운 RFC나 draft로 정당화해 온 전례를 지적하며, 이번에도 통합 테스트 부족과 책임 회피성 커뮤니케이션을 비판하는 목소리가 있다.[[news.hada](https://news.hada.io/topic?id=25981)]​

---

## 배경 지식​

## 1. DNS와 A/CNAME 레코드 기본

- DNS는 사람이 읽는 도메인 이름을 실제 서버의 IP 주소로 바꿔주는 인터넷의 전화번호부 같은 시스템이다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
- A 레코드는 `이름 → IPv4 주소`를 직접 가리키고, CNAME 레코드는 `이 이름은 다른 이름의 별칭(aliased name)`이라는 뜻이라, CNAME을 한 번 더 따라가 A/AAAA 레코드를 얻어야 최종 IP를 알 수 있다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

## 2. CNAME 체인과 “섞인 응답”의 순서

- 도메인 하나가 여러 단계의 CNAME을 거쳐 최종 IP를 찾는 경우가 흔하다. 예를 들어 `www.example.com → cdn.example.com → server.cdn-provider.com → 198.51.100.1`처럼 이름에서 이름, 마지막에 A 레코드로 이어지는 **체인**이 만들어진다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
- 캐시를 가진 리졸버(1.1.1.1 같은)는 이 체인 전체를 항상 처음부터 새로 조회하지 않고, 각 링크별 TTL을 보고 **만료된 부분만 다시 질의**한 뒤, 캐시된 레코드와 새로 가져온 레코드를 **하나의 응답 메시지로 합쳐서** 클라이언트에 돌려준다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

여기서 중요한 부분이 “합칠 때 레코드를 어떤 순서로 배열하느냐”이다.[[news.hada](https://news.hada.io/topic?id=25981)]​

- 기존 구현에서는 “CNAME 체인 → 최종 A/AAAA 레코드” 순으로, 즉 **CNAME들을 먼저, 그 다음에 주소 레코드들을 answer 섹션에 넣는 방식**을 썼다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
    - 요약하면:
        
        - 1단계: `www.example.com CNAME cdn.example.com`
            
        - 2단계: `cdn.example.com CNAME server.cdn-provider.com`
            
        - 3단계: `server.cdn-provider.com A 198.51.100.1`
            
        - 응답 순서: 1단계 CNAME → 2단계 CNAME → 마지막 A 레코드.[[news.hada](https://news.hada.io/topic?id=25981)]​
            
- 코드 변경 이후에는 내부 자료구조를 정리하는 과정에서 “먼저 A/AAAA 레코드를 넣고, 나중에 CNAME을 추가”하게 되어, **같은 레코드들이지만 순서가 ‘A/AAAA → CNAME’으로 뒤집힌 응답**이 나가는 경우가 생겼다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

문제는 일부 클라이언트가 이 순서에 **강하게 의존**하고 있었다는 점이다.[[news.hada](https://news.hada.io/topic?id=25981)]​

- glibc `getaddrinfo`는 응답의 answer 섹션을 **위에서부터 순차적으로 읽으며** “지금 보고 있는 이름”을 갱신한다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
    - 기대하는 패턴은:
        
        1. 먼저 `www.example.com CNAME cdn.example.com`을 보고 “아, 이제 cdn.example.com을 찾아야겠구나”라고 상태를 바꿈
            
        2. 그 다음 CNAME이나 A 레코드를 보면서 체인을 계속 따라가는 방식이다.[[news.hada](https://news.hada.io/topic?id=25981)]​
            
- 그런데 응답이 “A/AAAA 먼저, CNAME 뒤”로 섞여 나오면, glibc는 맨 앞에서 A 레코드를 보지만 **그 A가 어떤 이름에 대응하는지 자기 상태와 매칭을 못 해서** 결과를 비어 있다고 판단하거나, CNAME을 뒤에서 발견해도 이미 처리를 끝낸 상태가 되어 제대로 따라가지 못한다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

또한 CNAME이 여러 단계일 때, 체인 자체의 **내부 순서**까지 뒤섞이면 더 심각해진다.[[news.hada](https://news.hada.io/topic?id=25981)]​

- 예를 들어, 응답 안에
    
    - `cdn.example.com CNAME server.cdn-provider.com`
        
    - `www.example.com CNAME cdn.example.com`  
        이런 순서로 뒤집혀 있으면, 리졸버는 먼저 `cdn.example.com`에 대한 CNAME을 본 뒤, 나중에 `www.example.com`을 보게 되어 “내가 원래 찾던 이름에서 출발해 체인을 따라간다”는 흐름이 깨진다.[[news.hada](https://news.hada.io/topic?id=25981)]​
        
- 이때 순서에 의존하지 않고 전체를 한 번에 자료구조로 모아 탐색하는 리졸버(systemd-resolved처럼)는 문제 없이 “그래프 탐색”하듯 체인을 복원하지만, 단순히 “위에서 아래로 읽어 내려가는” 구현은 체인을 재구성하지 못해 실패한다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

요약하면, 같은 레코드라도 **캐시에서 꺼낸 것 + 새로 조회한 것**을 합치는 과정에서 “어느 쪽을 먼저 answer 섹션에 넣느냐”가 바뀌면, 일부 구현에게는 사실상 **“다른 형식의 프로토콜”처럼 보이는 수준의 변화**가 되어버렸다는 뜻이다.[[news.hada](https://news.hada.io/topic?id=25981)]​

## 3. 리졸버 종류와 glibc의 한계

- DNS 질의를 직접 날리는 프로그램은 많지 않고, 대부분 운영체제의 라이브러리(예: 리눅스 glibc의 `getaddrinfo`)를 통해 이름을 IP로 바꾼다. 이런 라이브러리는 보통 캐시도 없고, 응답을 고급 자료구조로 재정렬하지도 않는 **스텁 리졸버**에 속한다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
- glibc처럼 단순한 구현은 “응답을 순서대로 읽는다”는 전제 위에서 작성되었고, 레코드 순서를 무시하려면 별도 인덱스나 맵 자료구조를 두고 탐색해야 해서 코드 구조가 크게 바뀌어야 한다. 이 때문에 “순서에 의존하지 말라”는 이상과 “이미 수십 년 사용 중인 구현” 사이에 간극이 생겼다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

## 4. RFC 1034의 애매한 문구와 해석 차이

- DNS의 기본을 정의하는 RFC 1034는 “응답은 하나 이상의 CNAME으로 앞부분이 될 수 있다(possibly preface)”라고 적어, “CNAME이 앞에 올 수 있다”는 예시는 주지만, “반드시 앞에 와야 한다”는 식의 강한 규정을 넣지는 않았다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
- 또 다른 부분에서는 “같은 이름·타입·클래스의 레코드 집합(RRset)은 순서가 중요하지 않다”고 하면서, 예시로 A 레코드들만 다룬다. 글과 댓글에서는 이 문장을 “RRset 내부에만 적용되는 예외”로 봐야 하는데, Cloudflare는 이를 “응답 섹션 전체의 순서가 중요하지 않다”는 일반 규칙으로 과도하게 확장 해석했다고 비판한다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

즉, 표준 문서가 **MUST/SHOULD** 같은 규범적 단어 없이 서술형으로 쓰이던 시대의 산물이라,

- 어떤 구현은 “CNAME이 있으면 앞에 둬야 한다”에 가깝게 이해했고,
    
- 다른 구현은 “어떻게 섞여 있어도 상관없다”고 이해한 상태에서,  
    서로 다른 해석이 수십 년 쌓이다가 이번에 한 번 크게 충돌한 셈이다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

## 5. Cloudflare의 교훈과 조치

- Cloudflare는 1.1.1.1 코드에서 CNAME을 뒤에 넣도록 바꿨다가 glibc, Cisco 장비 등 순서 의존 구현이 대규모로 깨지는 장애를 겪고, CNAME을 **항상 먼저 배치하는 정책**으로 되돌렸다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    
- 동시에 “어느 쪽이 맞느냐” 논쟁을 줄이기 위해, DNS 응답 내 CNAME 처리 순서를 명확히 규정하는 Internet-Draft(draft-jabley-dnsop-ordered-answer-section)를 IETF에 제출해, 오래된 RFC의 모호함을 현대적인 규범 언어로 보완하려는 시도를 하고 있다.[[news.hada](https://news.hada.io/topic?id=25981)]​
    

이 배경을 알고 나면, 글 전체가 “DNS 응답 안에서 CNAME과 A 레코드가 **똑같이 들어 있어도**, 그 **배열 순서**를 바꾸는 것이 실제로는 API 변경에 가깝고, 오래된 클라이언트 구현과 표준 문구의 모호함이 겹치면 인터넷 규모 장애로 이어질 수 있다”는 이야기라는 점이 더 잘 보인다.[[news.hada](https://news.hada.io/topic?id=25981)]​