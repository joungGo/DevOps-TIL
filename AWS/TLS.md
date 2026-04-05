### TLS

TLS 는 **HTTPS 암호화** 입니다. 🔐

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