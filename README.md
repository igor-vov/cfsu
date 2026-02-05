# CloudFront Signed Resource 통합 연동 규격서

본 문서는 AWS CloudFront의 **Signed URL/Cookie (개별 파일, 스트리밍/폴더 접근)** 기능을 귀사의 서비스에 연동하기 위한 통합 인터페이스 명세 및 가이드입니다.

연동에 앞서 가장 먼저 키페어를 만들어야 합니다.

## 보안 키 페어 생성

### openssl 을 사용해서 private key(URL생성) 와 public key(URL검증) 를 생성할 수 있습니다.

```ssh
openssl genrsa -out private_key.pem 2048
openssl rsa -pubout -in private_key.pem -out public_key.pem
```

## CloudFront Signed URL/Cookie용 키 등록

서명된 URL을 만들기 위해 **퍼블릭 키(.pem)**를 업로드하고 **Key Group**을 생성하는 곳입니다.

* **바로가기 URL:**
**[CloudFront 퍼블릭 키 등록 페이지 바로가기](https://www.google.com/search?q=https://us-east-1.console.aws.amazon.com/cloudfront/v4/home%3Fregion%3Dus-east-1%23/publickey)**
*(CloudFront는 글로벌 서비스이므로 리전 상관없이 위 링크로 접속됩니다)*
* **메뉴 이동 경로:**
AWS 콘솔 접속 > **CloudFront** > 왼쪽 메뉴의 **공용 키(Public keys)** > **공용 키 생성(Create public key)**

#### 등록 후 필수 절차

키를 등록한 후에는 반드시 **[키 그룹(Key Groups)](https://www.google.com/search?q=https://us-east-1.console.aws.amazon.com/cloudfront/v4/home%3Fregion%3Dus-east-1%23/keygrouplist)** 메뉴로 이동하여, 방금 등록한 키를 포함하는 **그룹**을 만들어야 CloudFront 배포(Distribution) 설정에서 선택할 수 있습니다.

** 참고 (개발 시 주의사항):**
CloudFront Signed URL을 구현하실 때는 **1번 메뉴**에서 발급된 **`Key Pair ID` (예: K2JCJMDEHXQW...)**를 코드의 환경변수(`key_pair_id`)에 넣어서 사용해야 합니다.


## 1. 아키텍처 개요

보안상 서명(Signing) 작업은 반드시 백엔드 서버 간 통신을 통해 이루어져야 합니다.

1. **Client:** 콘텐츠 요청 시 귀사 백엔드로 인증 요청
2. **Server:** 유저 권한 확인 후 서명 생성 모듈 호출
3. **Response:**
* **URL 방식:** 서명된 Query String이 포함된 URL 반환
* **Cookie 방식:** `Set-Cookie` 헤더를 통해 브라우저에 인증 토큰 저장


4. **Access:** Client가 CloudFront로 리소스 요청 시 인증 통과

---

## 2. API 인터페이스 명세 (Interface Specification)

모든 요청은 귀사의 백엔드 서버에서 아래 규격에 맞춰 호출해야 합니다.

* **Method:** `POST`
* **Content-Type:** `application/json`
* **Auth Header:** `x-api-key: {ISSUED_API_KEY}`

### 2.1 Request Body

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `request_type` | String | **Yes** | `url` (단일 파일) 또는 `cookie` (스트리밍/폴더) |
| `resource_url` | String | **Yes** | 권한을 부여할 대상 URL (프로토콜 포함) |
| `expiry_seconds` | Integer | No | 유효 시간 (초 단위, Default: 300) |
| `client_ip` | String | No | (Optional) 특정 IP 제한이 필요한 경우 전송 |

**Sample Payload**
```http
POST /api/generate-signed-resource HTTP/1.1
Host: localhost:5000
Content-Type: application/json
x-api-key: YOUR_ISSUED_KEY

{
  "request_type": "cookie",
  "resource_url": "https://cdn.example.com/premium/content/*",
  "expiry_seconds": 3600,
  "client_ip": "203.0.113.10"
}

```

> **Note:** `cookie` 타입 사용 시 `resource_url`에 와일드카드(`*`)를 사용하여 하위 경로 전체에 권한을 부여하는 것을 권장합니다.

### 2.2 Response Body

**Case A: URL Type Response**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 150

{
  "status": "success",
  "mode": "url",
  "data": {
    "signed_url": "https://cdn.example.com/file.mp4?Policy=...&Signature=..."
  }
}

```

**Case B: Cookie Type Response**
Response Body에는 메타데이터만 포함되며, **핵심 인증 정보는 Response Header에 포함됩니다.**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: CloudFront-Policy=ey...; Domain=.example.com; Path=/; Secure; HttpOnly
Set-Cookie: CloudFront-Signature=...; Domain=.example.com; Path=/; Secure; HttpOnly
Set-Cookie: CloudFront-Key-Pair-Id=...; Domain=.example.com; Path=/; Secure; HttpOnly

{
  "status": "success",
  "mode": "cookie",
  "message": "Cookies set successfully"
}

```

* **Response Headers:**
* `Set-Cookie: CloudFront-Policy=...; Path=/; Domain=.example.com; Secure; HttpOnly`
* `Set-Cookie: CloudFront-Signature=...; ...`
* `Set-Cookie: CloudFront-Key-Pair-Id=...; ...`



---

## 3. Backend 구현 예제 (Python/Flask)

필수 라이브러리 설치
서명 생성에 필요한 암호화 라이브러리와 AWS SDK를 설치해야 합니다.

```bash
pip install cryptography boto3 Flask requests
```

URL 반환과 Cookie 설정을 단일 엔드포인트에서 처리하는 예제입니다.

```python
from flask import Flask, request, jsonify, make_response
import requests

app = Flask(__name__)

# Configuration
SIGNING_API_URL = "https://api.provider.com/generate-token"
API_KEY = "YOUR_ISSUED_KEY"
CDN_DOMAIN = ".your-service-domain.com" # 상위 도메인 설정 (필수)

@app.route('/api/auth/resource', methods=['POST'])
def authorize_resource():
    try:
        # 1. Request Parsing
        payload = request.json
        req_type = payload.get('request_type', 'url')
        
        # 2. Call Signing API
        headers = {'x-api-key': API_KEY, 'Content-Type': 'application/json'}
        res = requests.post(SIGNING_API_URL, json=payload, headers=headers)
        
        if res.status_code != 200:
            return jsonify({'error': 'Upstream signing failed'}), res.status_code
            
        auth_data = res.json().get('data', {})

        # 3. Response Handling
        if req_type == 'url':
            return jsonify({
                'status': 'success',
                'signed_url': auth_data.get('signed_url')
            })

        elif req_type == 'cookie':
            # Create Response with Cookies
            resp = make_response(jsonify({'status': 'success', 'mode': 'cookie'}))
            cookies = auth_data.get('cookies', {})
            
            for key, val in cookies.items():
                resp.set_cookie(
                    key, val,
                    domain=CDN_DOMAIN,
                    path='/',
                    secure=True,       # CloudFront 필수 요구사항
                    httponly=True,     # XSS 방지
                    samesite='None'    # Cross-site 설정 (상황에 따라 조절)
                )
            return resp

    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(port=5000, debug=True)

```

---

## 4. API 테스트 가이드 (Testing)

개발 단계에서 정상 동작을 검증하기 위한 방법입니다.

### 4.1 cURL을 이용한 테스트 (CLI)

터미널에서 아래 명령어로 응답을 확인합니다.

**URL 방식 테스트**

```bash
curl -X POST http://localhost:5000/api/auth/resource \
     -H "Content-Type: application/json" \
     -d '{
           "request_type": "url",
           "resource_url": "https://cdn.example.com/file.png"
         }'

```

* **성공 기준:** JSON 응답 내 `signed_url` 필드 확인.

**Cookie 방식 테스트 (Header 확인)**

```bash
curl -I -X POST http://localhost:5000/api/auth/resource \
     -H "Content-Type: application/json" \
     -d '{
           "request_type": "cookie",
           "resource_url": "https://cdn.example.com/videos/*"
         }'

```

* **성공 기준:** 응답 헤더에 `Set-Cookie: CloudFront-Policy=...` 등 3개의 쿠키가 출력되어야 함.

### 4.2 Postman 설정 가이드

GUI 툴인 Postman 사용 시 설정 값입니다.

1. **Method / URL:** `POST` `http://localhost:5000/api/auth/resource`
2. **Headers:**
* `Content-Type`: `application/json`


3. **Body (raw / JSON):**
```json
{
  "request_type": "cookie",
  "resource_url": "https://cdn.example.com/premium/*",
  "expiry_seconds": 3600
}

```


4. **검증 방법:**
* Send 버튼 클릭 후 하단 **Cookies** 탭 확인.
* `CloudFront-Policy`, `CloudFront-Signature`, `CloudFront-Key-Pair-Id` 3개 항목이 존재하면 정상입니다.



---

## 5. 트러블슈팅 (Troubleshooting)

연동 중 발생하는 주요 에러 코드 및 조치 사항입니다.

| Status | Error Code / Message | 원인 및 해결 방안 |
| --- | --- | --- |
| **403** | `AccessDenied` (CloudFront) | **URL 불일치:** 서명된 URL(`resource_url`)과 실제 접근 URL이 다름.|
| | | **Cookie Domain:** 쿠키의 `Domain` 속성이 CDN 도메인과 매칭되지 않음.|
| | | **프로토콜:** `http`로 접근 시도 (Signed Cookie는 `https` 권장). |
| **400** | `Missing Parameter` | 요청 JSON에 `resource_url` 누락 여부 확인. |
| **401** | `Unauthorized` | `x-api-key` 헤더 누락 또는 잘못된 키 사용. |
| **N/A** | **CORS Error** (Browser) | S3/CloudFront의 CORS 설정 확인. API 호출 시 `credentials: 'include'` 옵션 확인. |

---
