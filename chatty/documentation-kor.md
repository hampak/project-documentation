# 목차
1. [프로젝트 개요](#프로젝트-개요)
2. [API 문서](#api-문서)
3. [웹소켓 문서](#웹소켓-문서)
4. [앱 기능](#앱-기능)
5. [개발하면서 마주친 버그 및 에러사항](#개발하면서-마주친-버그-및-에러사항)
6. [프로젝트 개발하면서 배우고 느낀 점](#프로젝트-개발하면서-배우고-느낀-점)

# 프로젝트 개요
Chatty는 유저들이 실시간으로 채팅을 할 수 있는 어플리케이션입니다. 실시간 커뮤니케이션(real-time communication)이라는 개념 자체가 저에겐 매우 신기했으며 언젠가 이런 프로젝트를 개발하고 싶었습니다. Chatty를 개발하면서 저는 redis와 web sockets와 같은 기술을 많이 그리고 깊게 배울 수 있었습니다.

Chatty는 밑에 기재된 스택으로 개발되었습니다:
```json
{
  "express",
  "mongoose",
  "socket.io",
  "jsonwebtoken",
  "redis",
  "google-auth-library",
  "cookie",
  "react",
  "@tanstack/react-query",
  "react-hook-form",
  "react-router-dom",
  "tailwind",
  "zod",
  "zustand"
}
```

**Express.JS**를 선택한 이유는 좀 더 **Express**와 친숙해지고 익숙해지고 싶어서 골랐습니다. 메인 데이터베이스는 **MongoDB**를 선택했으며 데이터 읽기/쓰기가 많이 요구되는 데이터는 **redis(upstash)** 를 활용했습니다.

여기서 중요한 점 한 가지는 Kinde 혹은 Clerk와 같은 사용자 인증 서비스를 사용하지 않았다는 겁니다. 저는 **OAuth 2.0**을 깊게 다뤄보고 싶어서 제3 서비스를 활용하지 않았습니다.

프론트쪽은 리액트를 사용했습니다. 라우팅과 페이지 네비게이션은 **react router dom**을 활용했으며, 데이터 요청은 **tanstack query**를 활용했습니다. UI는 **tailwindcss**로 선택했습니다.

백앤드 서버는 **railway**에 배포되었으며 클라이언트는 **vercel**에 배포되어 있습니다.

해당 프로젝트를 개발하면서 저는 세계에서 가장 많이 사용되고 인기가 있는 스택에 대해서 더 깊게 공부하고 싶었습니다.

# API 문서
- [`/api/auth`](#apiauth)
- [`/api/user`](#apiuser)
- [`/api/friend`](#apifriend)
- [`/api/chat`](#apichat)

### `/api/auth`
- [`api/auth/google`](#apiauthgoogle)
- [`api/auth/google/callback`](#apiauthgooglecallback)
- [`api/auth/github`](#apiauthgithub)
- [`api/auth/github/callback`](#apiauthgithubcallback)
- [`api/auth/logout`](#apiauthlogout)
- [`api/auth/check-auth`](#apiauthcheck-auth)

#### `/api/auth/google`
**Method**: GET

유저가 "Start with Google" 버튼을 클릭했을 때 요청되는 API입니다. 해당 API가 호출되면, **google-auth-library** 패키지를 활용하여 **redirect url**를 생성하게 됩니다.

```ts
const authUrl = oauth2Client.generateAuthUrl({
  access_type: "offline",
  scope: 'https://www.googleapis.com/auth/userinfo.profile  openid ',
  prompt: "consent"
})
```

유저는 이후, **authUrl** 페이지로 이동합니다. 이 페이지에서 유저는 로그인할 구글 계정을 선택합니다.

이 뿐만 아니라 해당 API 주소는 현재 유저의 브라우저에 쿠키가 있는지 확인합니다. **user**라는 이름의 쿠키를 찾으면 서버로 보내지고, (JWT를 이용하여) 디코딩된 토큰이 데이터베이스에 있는 것과 일치하는지 혹은 유효한지 판단합니다. 유효하지 않다면 (예를 들어 유저가 임의로 쿠키값을 설정했다면) 유저는 다시 **authUrl** 페이지로 이동하게 됩니다. 만약, 유효한다면 이미 로그인이 되어 있는 상태이므로 **authUrl** 페이지로 이동하는 않고 대시보드 페이지로 이동합니다.

```ts title="/api/auth/google"
const token = req.cookies.user

if (token) {
  try {
    const decoded = jwt.verify(token, JWT_SCRET!)
    if (typeof decoded !== "string" && decoded.user._id) {
      User.findById(decoded.user_id).then(user => {
        if (user) {
          return res.redirect(`${CLIENT_URL}/dashboard`)
        } else {
          // authUrl을 생성하고 구글 계정 선택 화면으로 유저를 이동시킵니다
        }
      })
    }
  } catch (error) {
    // 에러 발생 시, authUrl을 생성하고 구글 계정 선택 화면으로 이동시킵니다
  }
} else {
  // authUrl을 생성하고 구글 계정 선택 화면으로 이동시킵니다
}
```

#### `/api/auth/google/callback`
**Method**: GET

유저가 사용할 구글 계정을 선택하면, 해당 API가 호출됩니다. 해당 API에는 구글로부터 유저별로 부여된 고유의 id인 **sub**, 유저의 이름과 프로파일 이미지 등을 추출할 수 있는 중요한 로직을 코딩했습니다.

해당 API에서 가장 중요한 로직 중 하나는 유저가 새롭게 가입하는 유저인지 아닌지 확인하는 겁니다. 이것은 위에서 언급된 **sub** 값으로 이미 저장된 유저가 있는지 확인하는 걸로 처리가 됩니다. 만약 데이터베이스에 유저가 존재한다면 새롭게 추가하지 않으며 존재하지 않다면 새롭게 추가합니다.

이렇게 모든 로직을 통과한 후, 유저의 데이터는 **user**라는 쿠키로 브라우저에 저장됩니다.

```ts
const token = jwt.sign({
  user_id,
  name,
  picture
},
  JWT_SECRET!,
  { expiresIn: "60m" }
)

res.cookie("user", token, {
  httpOnly: true,
  maxAge: 60 * 60 * 1000,
  // 다른 옵션 추가
})

res.redirect(`${CLIENT_URL}/dashboard`)
```

#### `/api/auth/github`
**Method**: GET

해당 API는 유저가 "Start with Github" 버튼을 클릭했을 때 요청됩니다. 이 주소는 깃허브 redirect URL을 돌려줍니다.

```ts
.get("/github", async (req, res) => {
  const redirectUrl = `https://github.com/login/oauth/authorize?client_id=${GITHUB_CLIENT_ID}&redirect_uri=http://localhost:8000/api/auth/github/callback&scope=user`

  res.redirect(redirectUrl)
})
```

