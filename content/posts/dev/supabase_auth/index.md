---
title: "supabase으로 매직링크 사용자 인증"
date: 2022-05-05
description: "supabase으로 매직링크 사용자 인증"
summary: "supabase으로 매직링크 사용자 인증"
categories: ["dev"]
tags: ["supabase", "auth"]
---

사이드로 만들 사이트 인증 쪽 하다가 배운 것 정리

## supabase란?

[https://supabase.com/](https://supabase.com/)

파이어베이스 대체제로 나온 서비스이다. 파이어베이스보다 좋은 점은 PostgreSQL을 데이터베이스로 사용이 가능하다. 파이어베이스는 db 쿼리가 좋지 않은 거로 아는데, 이 부분에선 PostgreSQL을 쓸 수 있는 supabase가 확실히 낫다. 그리고 셀프 호스팅으로 저렴하게 사용할 수 있어서 호감이다.

이 포스팅에서 다룰 인증은 파이어 베이스에도 있는 기능이다. 아마 사용방식도 비슷할 것 같다.

## 프로젝트 설정

[https://app.supabase.io/](https://app.supabase.io/)

supabase에서 프로젝트 생성 후, authentication 설정을 들어간다.

![supabase setting](./1.png)

사이트 url과 jwt expiry를 설정하자. jwt 만료시간은 accessToken에 적용된다. refreshToken은 왜 없는지 궁금했는데, jwt 토큰 형식이 아니었다.

```jsx
{
    "provider_token": null,
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJhdXRoZW50aWNhdGVkIiwiZXhwIjoxNjUxNzU1ODkyLCJzdWIiOiJjYjE5NDliNy0yNzJkLTQ2OWItYjRiYS01M2RhMDE4YTllMmIiLCJlbWFpbCI6InNvc28wMWRldkBnbWFpbC5jb20iLCJwaG9uZSI6IiIsImFwcF9tZXRhZGF0YSI6eyJwcm92aWRlciI6ImVtYWlsIiwicHJvdmlkZXJzIjpbImVtYWlsIl19LCJ1c2VyX21ldGFkYXRhIjp7fSwicm9sZSI6ImF1dGhlbnRpY2F0ZWQifQ.6wch2tYoc3ynLl_RxM7ZGFH4DrREOGDO8HyV-H2-T6o",
    "expires_in": 3600,
    "expires_at": 1651755893,
    "refresh_token": "YfHHAXSBlltxLoJ4kbXU5Q",
    "token_type": "bearer",
    "user": {
			...
    }
}
```

위 포맷으로 세션 정보를 가져온다. refresh 토큰은 jwt 토큰이 아닌, 그냥 키값이었다. 그리고 세션을 갱신할 때마다 refresh_token도 변경된다. 평소에 사용하던 jwt 토큰의 refresh 방식이 아니라 일회용 키값을 발급해주는 식인 것 같은데 만료가 되는지는 잘 모르겠다.

## 프론트엔드 설정

```tsx
import { createClient } from "@supabase/supabase-js";

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

export const supabase = createClient(supabaseUrl, supabaseAnonKey);
```

@supabase/supabase-js js 용 supabase 라이브러리를 설치 후, supabaseClient 인스턴스를 생성한다.

```tsx
await supabase.auth.signIn({ email });
```

그리고 생성한 supabase 인스턴스에 auth.signIn() 으로 로그인을 요청할 수 있다. 이메일만 설정하면 자동으로 매직 링크 로그인 방식으로 인지하고 이메일을 보낸다.

![supabase magic link](./2.png)

이메일의 기본 포맷은 위와 같다. 링크 주소는 [https://ycfqxoxuzltvyozzmbdt.supabase.co/auth/v1/verify?token=oseidldaadtnhomfwdme&type=magiclink&redirect_to=http://localhost:3000/](https://ycfqxoxuzltvyozzmbdt.supabase.co/auth/v1/verify?token=oseidldaadtnhomfwdme&type=magiclink&redirect_to=http://localhost:3000/)

인데, token이 리프레쉬 토큰이고, 액세스 토큰을 발급해서 redirect 주소로 쿼리스트링과 함께 보내주는 듯 하다.

```tsx
// 리프레쉬
supabase.auth.refreshSession();

// 로그아웃
supabase.auth.signOut();
```

리프레쉬와 로그아웃 호출은 위와 같이 메서드를 호출 하면 된다.

## axios 설정

![스크린샷 2022-05-05 오후 9.26.43.png](./3.png)

![스크린샷 2022-05-05 오후 9.34.52.png](./4.png)

supabase 대시보드에서 액세스 jwt 토큰의 시크릿 키를 제공해 주기 때문에, 독자적으로 accesskey verify를 할 수 있다. 그래서 나는 supabase를 인증서버로만 사용하고, 비즈니스 로직을 다룰 백엔드 서버는 따로 두려고 한다.

프론트에서 supabase의 액세스키를 헤더에 실어서 백엔드로 보내고, 백엔드에선 jwt를 verify하고 payload에 담긴 유저 id(sub)으로 로직을 수행한다.

```tsx
import axios, { Axios, AxiosError } from "axios";
import { supabase } from "@/config/supabaseClient";
import { Session, SupabaseClient } from "@supabase/supabase-js";

class API {
  api: Axios;
  supabase: SupabaseClient;

  constructor(supabase: SupabaseClient) {
    this.api = axios.create();
    this.supabase = supabase;

    this.setBaseURL();
    this.setAuthorization(this.supabase.auth.session());
    this.setInterceptor();
  }

  setBaseURL() {
    this.api.defaults.baseURL = import.meta.env.VITE_BASE_URL;
  }

  setAuthorization(session: Session | null) {
    const accessToken = session?.access_token;
    const authorization = accessToken ? `Bearer ${accessToken}` : "";
    this.api.defaults.headers.common["Authorization"] = authorization;
  }

  setInterceptor() {
    this.api.interceptors.response.use(
      (res) => res,
      async (error: AxiosError) => {
        const { config, response } = error;

        if (response?.status === 401 && this.supabase.auth.session()) {
          const accessToken = await this.refresh();

          if (accessToken) {
            return axios({
              ...config,
              headers: {
                Authorization: `Bearer ${accessToken}`,
              },
            });
          }
        }

        return Promise.reject(error);
      }
    );
  }

  async refresh() {
    const response = await this.supabase.auth.refreshSession();
    this.setAuthorization(this.supabase.auth.session());

    return response.data?.access_token;
  }

  logout() {
    this.supabase.auth.signOut();
  }
}

export const apiSetting = new API(supabase);
export const api = apiSetting.api;
```

내 백엔드에 http 요청을 보내기 위해 axios 인스턴스도 따로 설정했다.

로그인하면 setAuthorization 메서드로 헤더에 액세스토큰을 설정하고, 만약 백엔드에서 401 에러를 반환하면, 인터셉터에서 supabase에서 제공하는 refresh 메서드를 실행 후 재요청한다.

백엔드는 nestjs 다시 공부하고 추가 예정

## 후기

외부 서비스로 인증하니 너무 쉽고 편하게 구현할 수 있어서 좋았다. 전에 인증 서비스로 auth0가 유명하길래 봤는데, 그건 너무 비싸서 사용할 엄두가 안 났었다. supabase는 인증서버로만 사용한다면 사실상 평생 무료처럼 보인다.
앞으로 사이드프로젝트에서 인증은 무조건 이거 쓸 것 같다.
