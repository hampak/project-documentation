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
