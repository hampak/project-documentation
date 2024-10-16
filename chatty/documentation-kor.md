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
  // 다른 옵션 추가
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

#### `/api/auth/github/callback`
**Method**: GET

위 API에서 깃허브 redirect URL로 리다이랙팅된 후, 유저는 이 주소로 이동하게 됩니다. **GITHUB_CLIENT_ID**와 **GITHUB_CLIENT_SECRET**를 이용하여 access token을 받게 됩니다. 이 토큰을 통해 유저의 깃허브 정보 - 프로필 사진, 이름, 그리고 아이디를 얻습니다. **id**는 MongoDB에 저장됩니다.

```ts
const { code } = req.query;

if (!code) {
  return res.status(400).send("No code provided")
}

try {
  const tokenResponse = await axios.post(
    "https://github.com/login/oauth/access_token",
    {
      client_id: GITHUB_CLIENT_ID,
      client_secret: GITHUB_CLIENT_SECRET,
      code
    },
    {
      headers: {
        accept: "application/json"
      }
    }
  )

  const accessToken = tokenResponse.data.access_token

  const userResponse = await axios.get("https://api.github.com/user", {
    headers: {
      Authorization: `Bearer ${accessToken}`
    }
  })

  const { avatar_url, name, id } = userResponse.data
  ...
  } catch ...
```

이후 코드는 위의 `/api/auth/google/callback` API 로직과 비슷하게 흘러갑니다.

#### `/api/auth/logout`
**Method**: GET

해당 API는 유저를 로그아웃 시킵니다. 세션 정보가 담겨있는 쿠키를 브라우저와 redis 데이터베이스에서 삭제합니다. 이후, 유저는 대쉬보드 페이지로 이동됩니다.

```ts
.get("/logout", async (req, res) => {
    const token = await req.cookies.user
    const decoded = jwt.verify(token, JWT_SECRET!) as JwtPayload

    res.clearCookie("user")
    await redis.del(`sessionToken-${decoded.user_id}`)
    res.redirect(`${CLIENT_URL}`)
  })
```

#### `/api/auth/check-auth`
**Method**: GET

해당 API는 라우트 보호(route protection)를 위해 만들었습니다. 먼저, 유저 브라우저에 "user"라는 이름의 쿠키가 존재하는지 확인합니다. 만약 없다면, **401** 에러를 리턴하고 유저를 `/login` 페이지로 이동시킵니다.

만약 "user"라는 이름의 쿠키가 존재한다면, 데이터베이스에 일치하는 값이 있는지 확인합니다. 있다면 **200** 코드를 리턴하고 없다면 **401** 에러 코드를 리턴한 후, 유저를 `/login` 페이지로 이동시킵니다.

### `/api/user`
- [`/api/user`](#apiuser)

#### `/api/user`
**Method**: GET

해당 API는 현재 로그인된 유저의 정보를 불러오는데 사용됩니다. 세션 토큰이 유효한지 먼저 확인 한 후, 유효하다면 MongoDB에서 유저의 정보를 가져옵니다. 불러오는 데이터는 이렇습니다:

- id - 유저를 구분 짓는 고유의 아이디 (MongoDB가 생성함)
- name - 유저의 이름 (구글 혹은 깃허브에 설정된 이름)
- picture - 유저의 프로파일 사진
- online - 유저의 접속 상태 (**해당 값은 redis에서 불러옵니다**)
- userTag - 사용자가 서로 친추 추가를 위해 사용되는 값입니다.

추가적으로, 토큰에 이상이 있다면 서버는 **401** 에러코드를 리턴합니다.

### `/api/friend`
- [`/api/friend`](#apifriend)
- [`/api/friend/add-friend`](#apifriendadd-friend)
- `/api/friend/delete-friend` **빌드 전 단계**

#### `/api/friend`
**Method**: GET

해당 API는 현재 로그인된 유저의 친구를 불러옵니다. 클라이언트에서 **userId** 값을 받은 후, 데이터베이스에서 해당 아이디를 가진 유저의 친구를 불러옵니다. 만약, **userId**가 존재하지 않는다면 **401** 에러코드를 리턴합니다.

만약 있다면, 해당 데이터를 리턴합니다.

- _id - MongoDB가 생성한 유저별 고유의 아이디
- name - 친구의 이름
- image - 친구의 프로파일 사진
- userTag - 친구의 유저테그

이후, 이 값은 **friends**라는 어레이에 저장된 후, 클라리언트로 보냅니다.

```ts
const friendsList = await User.find({
  _id: { $in: user.friends }
}).select("_id name image userTag")

const friends = friendsList.map(friend => ({
  userId: friend._id.toString(),
  name: friend.name,
  image: friend.image,
  userTag: friend.userTag
}))
```

#### `/api/friend/add-friend`
**Method**: POST

해당 API는 친구추가를 담당합니다. 클라이언트로부터 **friendUserTag**와 **userId** 값을 받습니다. 먼저, 추가하려는 친구와 이미 친구인지 확이합니다. 이미 친구이면 클라이언트에 에러메세지를 리턴합니다. 또한, 유저가 자기자신을 친구로 추가하려는지 확인합니다.

이러한 체크가 통과된다면 해당 유저의 "friends" 필드에 추가하려는 친구의 아이디를 넣고 **반대로 똑같이 친구의 "friends" 필드에 현재 유저를 추가합니다.**

Redis에서도 친구 목록을 업데이트 합니다. 웹 소켓 서버가 추후에 빠르게 데이터를 쿼리하기 위함입니다.

친구가 성공적으로 데이터베이스에 추가되었다면 클라이언트로 해당 데이터를 리턴합니다.

- friendUserTag - 추가된 친구의 유저테그
- friendName - 추가된 친구의 이름
- friendId - 추가된 친구의 아이디

### `/api/chat`
- [`/api/chat/chat-list`](#apichatchat-list)
- [`/api/chat/create-chat`](#apichatcreate-chat)
- [`/api/chat/chat-info`](#apichatchat-info)

#### `api/chat/chat-list`
**Method**: GET

해당 API는 로그인된 유저가 참여하고 있는 챗 목록을 불러옵니다. 클라이언트로부터 받은 "userId"가 유효한지 확인 한 후, API는 필요한 데이터를 데이터베이스로부터 쿼리합니다.

```ts
const data = await ChatRoom.find({
  participants: { $elemMatch: { participantId: userId } }
})
```

데이터는 매핑이 되어서 **chatRooms**라는 어레이로 반환되고, 클라이언트로 리턴됩니다.

중요한 로직은 `.map()` 내에서 처리됩니다. 먼저, 각 채팅방마다 마지막으로 보내진 메세지를 추출합니다 (해당 메세지는 클라이언트에 사용됩니다). Redis 내에서 해당 데이터를 쿼리합니다.

```ts
const lastMessageRaw = await redis.zrange(`messages-${room._id}`, -1, -1)
lastMessage = lastMessageRaw[0] ? JSON.parse(lastMessageRaw[0]) : ""
```

이후, 채팅방 참여자들의 이름이 추출됩니다. 현재 로그인된 유저의 이름은 클라리언트로 보내지 않습니다.

```ts
const allParticipants = room.room_title.split("|").map(p => p.trim())
const friendName = allParticipants.find(p => p !== name)
```

이후, "last seen" 값도 추출합니다. 이것은 redis에서 쿼리합니다.

```ts
const lastSeenTimestamp = await redis.get(`last_seen-${userId}-${chatRoomId}`)
```

해당 코드는 **lastSeenTimestamp** 값이 존재하는지 확인합니다. 존재하지 않는다면 유저가 채팅방에 접속을 한 적이 없다는 겁니다. 이러기 위해서는 유저의 친구가 유저를 채팅방에 추가하고 메세지를 보낼 때 유저가 로그인하지 않았을 때입니다. 이런 경우엔 **unreadMessagesCount** 값을 채팅방 **전체** 메세지의 수로 설정합니다 (가장 처음으로 보낸 것 부터 마지막까지).

```ts
let unreadMessagesCount = 0;

if (!lastSeenTimestamp) {
  const allMessages = await redis.zrange(`messages-${chatRoomId}`, 0, -1);
  const unreadMessages = allMessages.filter(message => {
    const parsedMessage = JSON.parse(message)
    return parsedMessage.senderId !== currentUserId
  })
  unreadMessagesCount = unreadMessages.length;
} else ...
```

**unreadMessagesCount** 값을 추출하면서 현재 로그인된 유저가 보낸 메세지는 제외시킵니다. 유저가 채팅방에 접속을 한 적이 없어서 이럴 필요는 없겠지만 혹시나 모를 상황에 대비해 추가했습니다.

만약 **lastSeenTimestamp** 값이 있다면 유저가 톡방에 최소 한 번은 접속을 했다는 겁니다. 이런 경우, 유저가 마지막으로 접속했던 시간 **이후**로 저장된 메세지를 불러옵니다. 메세지가 이미 redis에 순서대로 저장이 되었기 때문에 **score**값을 이용해서 쉽게 불러올 수 있습니다.

```ts
else {
  // 유저가 마지막으로 접속했던 시간 이후로 보내진 메세지를 불러옵니다.
  const messagesAfterLastSeen = await redis.zrangebyscore(`messages-${chatRoomId}`, lastSeenTimestamp, "+inf")
  const unreadMessages = messagesAfterLastSeen.filter(message => {
    const parsedMessage = JSON.parse(message);
    return parsedMessage.senderId !== currentUserId
  })
  unreadMessagesCount = unreadMessages.length
}
```

이 모든 것을 마치고 난 후, 클라이언트에게 다음과 같은 데이터를 반환합니다.

- id - 채팅방 고유의 아이디
- createdAt - 채팅방이 생성된 날짜와 시간
- title - 채팅방의 "제목" 혹은 "이름". **friendName**값으로 설정됩니다.
- participants - 현재 채팅방에 참여하고 있는 유저들의 아이디와 프로파일 사진이 담겨있습니다.
- lastMessage - 클라이언트의 UI에 보여줄 마지막 메세지 값입니다.
- unreadMessagesCount - 현재 접속한 유저가 확인하지 않은 메세지의 개수입니다.

#### `api/chat/create-chat`
**Method**: POST

해당 API는 채팅방을 생성합니다. 서버는 클라이언트로부터 참여자들의 데이터를 받고 채팅방을 만듭니다.

코드는 클라이언트로부터 유효한 데이터를 받았는지 확인합니다 (예를 들어 현재 유저를 포함한 두 명 이상의 유저의 정보가 전달 되었는지 등). 이후, 코드는 1대1 채팅을 생성해야 하는지 단체채팅을 생성해야 하는지 판단합니다. 전자일 경우, (이미 1대1 채팅방이 존재하는지 확인 한 후) 채팅방을 생성합니다.

```ts
const chatroomAlreadyExists = await ChatRoom.findOne({
  participants: {
    $all: [
      { $elemMatch: { participantId: friendData[0].friendId } },
      { $elemMatch: { participantId: currentUserId } }
    ]
  },
  $expr: { $eq: [{ $size: "$participants" }, 2] }
})

if (chatroomAlreadyExists) {
  return res.redirect(`CLIENT_URL/dashboard/chat/${chatroomAlreadyExists._id}`)
}

const chatRoom = new ChatRoom({
  room_title: `${currentUserName}, ${friendData[0].friendName}`,
  participants: [
    { participantId: currentUserId, participantPicture: currentUserPicture },
    { participantId: friendData[0].friendId, participantPicture: friendData[0].friendPicture }
  ]
})

await chatRoom.save()
```

단체 채팅방을 생성하려고 하면 이미 같은 멤버들로 구성된 채팅방이 존재하는지 확인하지 **않습니다**. 이것은 일부러 이렇게 로직을 해놓은 겁니다. 예를 들어 User A, User B, 그리고 User C가 된 채팅방이 있는데 이 멤버들로 또 만드려고 하면 만들 수 있습니다.

```ts
const userNames = friendData.map((friend: Friend) => friend.friendName)
const roomTitle = `${currentUserName}, ${userNames.join(", ")}`

const participants = [
  { participantId: currentUserId, participantPicture: currentUserPicture },
  ...friendData.map((friend: Friend) => ({
    participantId: friend.friendId,
    participantPicture: friend.friendPicture
  }))
]

const chatRoom = new ChatRoom({
  room_title: roomTitle,
  participants: participants,
})

await chatRoom.save()
```




