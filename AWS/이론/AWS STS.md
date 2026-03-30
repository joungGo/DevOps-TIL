![Image](https://miro.medium.com/1%2ARNmKzxQesaeBs_oWan7kWg.jpeg)

![Image](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2024/11/12/01-Console-home-previous-1.png)

![Image](https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2018/01/21/AR_Diagram_010418.png)

![Image](https://awsfundamentals.com/assets/blog/aws-iam-roles-terms-concepts-and-examples/aws-iam-infographic.webp)

![Image](https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/07/14/ACM-PCA-Multi-Account-2020-ForSocial-400x800-1.jpg)

## AWS STS

AWS STS(AWS Security Token Service)는 Amazon Web Services(AWS)에서 제공하는 보안 토큰 관리 서비스로, 임시 보안 자격 증명(temporary security credentials)을 생성하여 사용자, 애플리케이션 또는 서비스가 AWS 리소스에 안전하게 접근할 수 있도록 지원한다. 이는 장기 액세스 키 사용을 줄이고, 권한 위임을 보다 세밀하게 제어할 수 있게 해 클라우드 보안성을 높인다.

### 주요 정보

- **제공 주체:** Amazon Web Services
    
- **출시 연도:** 2010년대 초
    
- **핵심 기능:** 임시 자격 증명 발급 및 교차 계정 액세스 제어
    
- **통합 서비스:** AWS IAM, EC2, S3, Lambda 등
    
- **보안 모델:** 최소 권한 원칙(least privilege)에 기반
    

### 작동 방식

STS는 사용자나 서비스가 신뢰 관계를 설정한 역할(role)을 요청하면, 일정 시간 동안만 유효한 액세스 키 ID, 시크릿 액세스 키, 세션 토큰을 발급한다. 이를 통해 사용자는 고정 자격 증명 없이 AWS API 호출을 수행할 수 있다. 자격 증명의 수명은 몇 분에서 몇 시간까지 설정할 수 있으며, 자동으로 만료되어 보안 리스크를 최소화한다.

### 주요 기능과 사용 시나리오

![Image](https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2016/11/16/Diagram_KZ_111616_d.png)

![Image](https://docs.aws.amazon.com/images/IAM/latest/UserGuide/images/saml-based-federation-diagram.png)

![Image](https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/09/29/IAM-users-YubiKey-ForSocial-1024x512.jpg)

![Image](https://media2.dev.to/dynamic/image/width%3D1280%2Cheight%3D720%2Cfit%3Dcover%2Cgravity%3Dauto%2Cformat%3Dauto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Fsefbtvjmwjuzgjpplkbx.png)

![Image](https://www.packetmischief.ca/2023/02/26/how-i-use-mfa-with-the-aws-cli/images/step2-retrieve-temp-credentials-using-mfa.png)

- **AssumeRole:** 다른 AWS 계정이나 서비스의 역할을 임시로 맡아 접근.
    
- **GetFederationToken:** 외부 사용자 또는 연합 사용자(federated user)를 위한 단기 자격 증명 제공.
    
- **AssumeRoleWithWebIdentity:** SAML 또는 OpenID Connect를 통한 싱글 사인온 지원.
    
- **AssumeRoleWithSAML:** 기업 인증 시스템과의 통합.
    

### 보안 및 관리 측면

STS는 IAM 정책, 역할 신뢰 정책, 세션 지속 시간 설정 등을 조합해 정교한 접근 제어를 가능하게 한다. 특히 다단계 인증(MFA)을 요구하거나 특정 IP 또는 네트워크 범위에서만 자격 증명 사용을 허용함으로써, 자격 증명 탈취 위험을 줄일 수 있다. 이를 통해 대규모 조직과 멀티 계정 환경에서의 보안 관리 효율을 높인다.