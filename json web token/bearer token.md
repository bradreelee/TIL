# Bearer Token

웹에서 인증과 인가를 다루다 보면, HTTP 요청에 다음과 같은 헤더를 넣는 일은 빈번한다.

```
Authorization: "Bearer ${token}"
```

이는 카카오가 제공하는 [카카오로그인 기능](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#logout-info)의 다양한 API 요청에서도 필수 헤더로 요구된다.

인증/인가를 공부하던 중 이 bearer의 의미에 대해 궁금해져서, 그에 대해 찾아본 내용을 정리해보았다.

## Bearer Token의 정의

Bearer token은 다음과 같은 정의를 가진다:

> A security token with the property that any party in possession of the token (a "bearer") can use the token in any way that any other party in possession of it can. Using a bearer token does not require a bearer to prove possession of cryptographic key material (proof-of-possession). 출처: [RFC6750](https://www.rfc-editor.org/rfc/rfc6750#section-1.2)

즉, 내가 이해한 바로는, 다음과 같은 상황에서 사용되는 token들을 *Bearer Token*이라고 할 수 있다:

> **_토큰을 소유한 사람 모두 같은 권리를 행사할 수 있을 때_**

이는 사실 "소유자" 라는 bearer의 사전적의미와 어느정도 부합하는 정의인 것 같다.

## 클라이언트가 token을 resource server에 보내는 방법

[RFC6750](https://www.rfc-editor.org/rfc/rfc6750#section-2.1)에 의하면 클라이언트 입장에서 resource server에 bearer access token을 보내는 방식은 3가지가 있다. **_이중에 하나의 방식만 사용하라고 한다._**

1. Authorization Request Header Field 이용
2. Request body 이용
3. URI query parameter에 access token 담아 보내기

3번 방식은 보안적 이슈 때문에 1,2번 방식이 불가능할 때만 사용하라고 RFC6750에 명시되있다.

2번방법은 사용시 지켜야될 요구사항이 너무 많고, 브라우저가 Authorization header를 이용할 수 없을시 사용하라고 한다.

**_그리고 결정적으로, 상황만 된다면, 클라이언트는 가능한 1번방식을 이용하여 resource server에 bearer access token을 전송해야한다고 명시되있다._**

## 서버가 client에게 bearer token을 보내주는 방법

서버가 client에게 bearer token 발급하여 보내줄 때에는, http response body에 넣어서 보내준다고 한다:

```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
    "access_token":"mF_9.B5f-4.1JqM",
    "token_type":"Bearer",
    "expires_in":3600,
    "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA"
}
```

출처: [Example Access Token Response](https://www.rfc-editor.org/rfc/rfc6750#section-4)

## 어떤 종류의 token들이 bearer token인가?

### access token

[위에서](#bearer-token의-정의) _토큰을 가지고 있는 사람 모두 같은 권리를 행사할 수 있다면_ 그 토큰은 bearer token이라고 할 수 있다고 하였다.

예를 들어 구체적으로 설명하자면, 카카오로그인 기능은 OAuth2.0 프로토콜을 기반으로 서비스 제공자에게 [access token](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#request-token)을 발급해준다.

서비스제공자는 이제 이 access token을 가지고 카카오로그인을 실행한 유저 대신에 카카오플랫폼 측에 유저관련 정보를 요청할수 있다. 그런데 카카오로그인이 제공하는 몇몇 기능(API)들은, 오로지 이 access token을 http요청에 담아보내는 것만으로도 이용이 가능하다.

**_즉, 만약 토큰이 제3자에게 탈취당한다면 제 3자도 똑같이 카카오 로그인이 제공하는 기능들을 이용할 수 있을 것이다. 이게 바로 "토큰을 소유한 사람 모두 같은 권리를 행사할 수 있을 때" 이며, 그래서 이 access token을 bearer token이라고 할 수 있는 것이다._**

### JWT

위와 마찬가지의 이유로 Json Web Token도 bearer token으로 취급할 수 있다고 생각한다 _(뇌피셜)_.

## 그냥 bearer token을 쿠키에 담아보내면 안되?

[RFC6750](https://www.rfc-editor.org/rfc/rfc6750#section-5.3)에서는 대놓고 bearer token을 쿠키에 저장하지 말라고 명시한다. 이는 쿠키의 보안적 이슈 때문이다. bearer token을 쿠키에 저장하고 싶다면, *cross-site request forgery*에 대한 대응방식을 갖추라고 한다.

### 카카오로그인 기능의 OIDC를 통한 인증과 관련하여...

카카오로그인기능은 OAuth 뿐만이 아니라 [OIDC를 활용하여 인증 기능](https://developers.kakao.com/docs/latest/ko/kakaologin/common#oidc)을 제공한다. 이 과정에서 사용자 인증의 기능을 수행하는 *id_token*이라는 것이 발급된다.

나는 이 id_token을 브라우저의 쿠키에 저장하여 사용하였다. [카카오 측에서도 id_token을 쿠키에 담아 사용하라고](https://devtalk.kakao.com/t/id-token/123122) 했기 때문이다.

하지만 RFC6750을 읽어본 이후로는, bearer token을 쿠키에 담아 사용하기 위해서는 보안에 신경써야한다는 점을 배웠다.

## 결론

- 클라이언트에서 token을 서버에 보낼 때, 그 토큰이 *bearer token*의 정의에 부합한다면, `Authentication: "Bearer ${token}"` 헤더를 사용하자.
- bearer token을 굳이 쿠키에 저장하고 싶다면 쿠키 보안에 신경써야한다.

(나는 여태까지 "클라이언트"라고 하면 항상 브라우저를 생각했다. 하지만 이 글에서 "클라이언트"는 "OAuth를 제공하는 서버에 요청을 보내는 주체"이며, 나는 그 주체를 "서비스를 제공하는 서버"로 염두해두었다.)

# 참고 문헌

- https://www.rfc-editor.org/rfc/rfc6750#section-2.1
- https://velog.io/@hyex/HTTP-Authorization-header%EC%97%90-Bearer%EC%99%80-jwt-%EC%A4%91-%EB%AC%B4%EC%97%87%EC%9D%84-%EC%82%AC%EC%9A%A9%ED%95%A0%EA%B9%8C
