# cookie를 통한 jwt를 사용

![인증 프로세스](https://user-images.githubusercontent.com/52027534/84857751-b4a41a00-b0a4-11ea-94b9-53d2a0ccd5e9.png)

## 인증 프로세스

- 최초 진입점을 통해 인증 토큰 A를 발급 받는다.
- A토큰을 들고 사용자 인증을 받아 B 토큰을 발급 받는다.
- B토큰은 시스템 내에서 매 요청마다 Header에 실어 보내 자동으로 인증받도록 한다.
- 인증과정에서 토큰이 만료되었으면 공통 핸들러에서 재 발급을 받거나 최초 인증과정으로 보내버린다.

## 토큰 A - JWT

사용자 인증을 위한 기본 토큰. 네트워크 내에나 특정 URL, 앱, 사용자 IP등을 토대로 시스템을 이용 가능한지에 대해 발급해준다.  
토큰 A는 특정 상태값을 가지고 있으며, 이 시나리오에서는 jwt를 쿠키로 생성해준다고 가정한다.

- httpOnly : 스크립트를 통해 접근할 수 없으며, 이 값은 request의 withCredential : true 옵션을 통해 자동으로 header에 전송 된다.
- secure : https를 통해서만 사용될 수 있도록 제한을 둔다
- domain : 해당 도메인을 통해서만 사용될 수 있도록 제한을 둔다
- API 호출시 withCredentials 옵션 예시
```javascript
const apiCall = axios.create({
  withCredentials: true,
  headers: {
    'Accept' : 'application/json',
    'Content-Type' : 'x-www-form-urlencoded'
  }
});
```

## 토큰 B - JWT

토큰 A를 들고 사용자 인증을 받는데, 몇 가지 제약이 있다.

- 토큰 A는 외부에서 제어를 하지 못 하도록 httpOnly 옵션이 있으며, request의 헤더에 자동으로 전송된다.
- 발급받은 토큰 B도 cookie에 저장한다.
- 토큰 B는 이제 인증 외의 모든 요청의 헤더에 Authorization: Bearer {token} 형식으로 자동으로 등록하도록 한다.
- 이미 스크립트 내에서 제어를 하는 값이므로 httpOnly 옵션은 크게 의미가 없다.
- Authorization 헤더 예시
```javascript
const userToken = 'userTokenSampler' // 이 부분에 jwt가 담긴다.
const apiCall = axios.create({
  headers: {
    'Accept' : 'application/json',
    'Content-Type' : 'x-www-form-urlencoded',
    'Authorization' : `Bearer ${userToken}`
  }
});
```

## 인증 실패

토큰 B를 통해 매 요청을 처리하는 도중 권한 오류가 발생하면 공통 핸들러를 통해 에러를 처리한다.  
3가지 시나리오를 통해 처리하는데, 하나는 refresh 토큰을 통한 재발급이 있으며,  
다른 하나는 인증 프로세스 최초지점으로 돌아가서 재 인증을 거치는 방식이다.  
마지막으로 사용자를 로그아웃 시키거나, 안내를 통해 시스템을 종료하게 하는 방법이다.

- refresh 토큰 : 사용자 토큰을 받을때 인증용 토큰과 인증용 토큰 재발급을 위한 refresh 토큰을 같이 받을 수 있다. refresh 토큰의 사용은 복잡한 시스템에서 사용자의 인증 정보를 유지시켜줘야 할때 유용한 방식이지만, 복잡성이 매우 높아지고, 토큰 탈취의 위험에 좀 더 노출되어 있다.
- 최초지점 이동 : 시스템 최초 진입시 항상 재인증을 받게 하는 경우에 유용하다. 토큰 A를 받는 최초 인증이나, B토큰 발급시 에러가 생길경우에 대한 시나리오를 충분히 대비해야 한다.
- 로그아웃 : 시스템이 아주 간편하며, 재진입에 부담이 없을때 사용하기 유용한 시나리오다.
