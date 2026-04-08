### TLS

 **TLS**는 인터넷 통신을 **암호화해서 안전하게 만드는 보안 프로토콜**이에요. 🔐

- 정식 이름: **Transport Layer Security**
- 우리가 흔히 보는 **HTTPS**가 바로 TLS 위에서 동작함
- 예전에는 **SSL**을 썼지만 지금은 TLS가 표준

- HTTP → 평문
    
- HTTPS → TLS 적용 (암호화)
    

Ingress 에서 TLS 설정 예:

```yaml
tls:
  - hosts:
      - url-shortener.local
    secretName: url-shortener-tls
```

역할:

- HTTPS 인증서 적용
    
- 브라우저 ↔ Ingress 암호화
    
- 비밀번호/토큰 보호
    

흐름:

```
Browser (HTTPS)
      ↓
TLS 종료 (Ingress)
      ↓
Service (HTTP)
      ↓
API Pod
```

정리:

- `url-shortener.local` → 호스트 이름 ✔️
    
- TLS → HTTPS 암호화 설정 ✔️