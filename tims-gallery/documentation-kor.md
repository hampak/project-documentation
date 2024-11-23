# 목차
1. [프로젝트 개요](#프로젝트-개요)
2. [API 문서](#api-문서)
3. [앱 기능](#앱-기능)
4. [개발하면서 마주친 버그 및 에러사항](#개발하면서-마주친-버그-및-에러사항)
5. [프로젝트 개발하면서 배우고 느낀 점](#프로젝트-개발하면서-배우고-느낀-점)

# 프로젝트 개요

Tim's Gallery는 제가 찍은 사진을 업로드하는 온라인 사진 갤러리입니다. 저는 디지털과 필름 사진을 찍는 것을 매우 좋아합니다. 그렇기 때문에 온라인 사진 갤러리를 한 번 만들어보고 싶어서 이렇게 만들게 되었습니다.

저는 (저만 엑세스하여 사진을 업로드 할 수 있는) 어드민 페이지와 실제 갤러리 페이지를 만들었습니다. 이 문서는 어드민 페이지와 갤러리 페이지 둘 모두에 대한 문서입니다.

어드민 페이지는 아래와 같은 기술을 사용했습니다.

```json
{
  "nextjs",
  "typescript",
  "aws-s3",
  "@tanstack/react-query",
  "tailwind",
  "zod",
  "react-hook-form",
  "drizzle-orm",
  "lucia",
  "blocknote"
}
```

**NextJS**는 사용하기 편리해서 선택했습니다. 찍은 사진을 업로드 할 스토리지는 아마존의 **S3**를 사용했습니다. 클라이언트 데이터 요청과 패칭은 **tanstack query**를 사용했으며 **zod**를 이용하여 리액트 폼(form)의 타입 체크를 했습니다. **Drizzle orm**를 사용하여 데이터베이스 읽기/쓰기 과정을 편리화 하였고 사용자 로그인 및 인증은 (2025년 3월에 곧 관리가 중단 될)[https://github.com/lucia-auth/lucia/discussions/1714] **lucia**를 활용했습니다. 스타일링은 **tailwindcss**를 활용했으며 텍스트 에디터로는 **blocknote**를 활용했습니다. 이는 노션의 블럭 에디터처럼 유저에게 높은 자유도를 제공해주는 라이브러리이기 때문입니다.

해당 갤러리를 빌드 하면서 가장 큰 난관은 **react hook form**를 사용하는 것이었습니다. 하나의 게시물(post)에 여러개의 사진(photo), 그리고 각 사진마다 갖고 있는 제목, 설명, 장비 상세 내역 등이 포함되어 있기 때문입니다. 하지만 복잡한 구조의 폼을 요한 기능을 구현했으며 문서 후반 부분에 설명을 해놨으니 참고하시면 됩니다.

샐러리 페이지는 해당 기술을 사용했습니다.

```json
{
  "nextjs",
  "typescript",
  "drizzle-orm",
  "tailwindcss",
  "blocknote",
  "@tanstack/react-query"
}
```

어드민 페이지와 비교했을 때 유사하며 더욱 간단합니다.

# API 문서

API 문서 섹션은 어드민 페이지와 갤러리 페이지로 구분 되어 있습니다. 아래 목록의 아이템을 클릭하여 원하시거나 궁금하신 부분으로 바로 이동하실 수 있습니다.

- [어드민 페이지 API](#어드민-페이지-api)
- [갤러리 페이지 API](#갤러리-페이지-api)

## 어드민 페이지 API
- [사용자 인증](#사용자-인증)
- [컨텐츠](#컨텐츠)

### 사용자 인증
- [sign-up-action](#sign-up-action)
- [sign-in-action](#sign-in-action)
- [sign-out-action](#sign-out-action)

#### `sign-up-action`
어드민 페이지를 사용하는 사람은 저 혼자이지만 유저 등록 절차에 대한 API를 설명해야 할 거 같아서 해당 서버 엑션 설명을 추가했습니다. 저 혼자만 사용하는 것이어서 유저 등록 기능을 구현하는 것이 조금 이상했지만 **lucia auth**와 **drizzle orm**를 이용하여 유저 등록 기능을 구현하고 싶었습니다.

**Drizzle orm**과 **lucia**를 사용한 이유는 제 자신에게 도전을 하고 싶어서 입니다. 해당 갤러리를 빌드할 시기에 **lucia**와 **drizzle orm**를 이용한 유저 인증에 관한 영상이나 게시글 등이 없었기 때문입니다. 하지만, 이를 한 번 도전하고 싶었습니다.

먼저, 클라이언트에서 서버 엑션으로 필요한 필드값을 보냅니다. 최대한 간단하게 구현하기 위해 필요한 필드값은 **username**과 **password**로 등록 기능을 구현했습니다.

```tsx
const onSubmit = (values: z.infer<typeof RegisterUserSchema>) => {
  startTransition(() => {
    signUpAction(values.username, values.password)
      .then((result) => {
        ...
      })
  })
}
```

해당 필드값은 서버 엑션으로 전송됩니다.

```ts
export async function signUpAction(username: string, password: string) {
  try {
    const user = await registerUserUseCase(username, password)
  } catch (err) {
    ...
  }
  return redirect("/")
}
```

서버 엑션 내에서는 `registerUserUseCase`라는 함수 안에서 사용됩니다. 해당 함수는 다음과 같은 로직을 따릅니다.

1. 먼저, 클라이언트로부터 받은 유저네임이 데이터베이스에 이미 있는지 확인합니다. 있다면 에러를 반환합니다.
2. 위의 체크를 통과하면 다음 단계로 넘어갑니다. 이 단계에서는 데이터베이스에 저장할 **salt**와 **hash**를 생성합니다. 이 값들은 인증 과정에서 사용될 값들입니다.
3. 이후, 새로운 유저를 데이터베이스에 추가합니다.

```ts
export async function registerUserUseCase(username: string, password: string) {

  const existingUser = await db.query.userTable.findFirst({
    where: eq(userTable.username, username)
  })

  if (existingUser) {
    return {
      errors: "Username already in use"
    }
  }

  ...
}
```

유저네임이 존재하는지 확인합니다.

이후, **salt**와 **hash**를 생성합니다.

```ts
const salt = crypto.randomBytes(128).toString("base64")
const hash = await hashPassword(password, salt)
const user = await db
  .insert(userTable)
  .values({
    hashedPassword: hash,
    username,
    salt,
    id: uuidv4(),
  })
  .returning()

return user
```

salt는 노드의 **crypto**라는 모듈을 활용하여 생성합니다. 이후, **hashed password**를 생성합니다. 이는 `hashPassword`라는 함수에 **password**와 **salt**를 보내서 생성하는데, 로직은 다음과 같습니다.

```ts
async function hashPassword(plainTextPassword: string, salt: string) {
  return new Promise<string>((resolve, reject) => {
    crypto.pbkdf2(
      plainTextPassword,
      salt,
      10000,
      64,
      "sha512",
      (err, derivedKey) => {
        if (err) reject(err)
        resolve(derivedKey.toString("hex"))
      }
    )
  })
}
```

**salt**와 **hash**를 성공적으로 생성한 후, 데이터베이스에 저장합니다.

마지막으로, **user**를 반환하고 유저를 메인 페이지로 이동시킵니다.

#### `sign-in-action`

해당 서버 엑션은 유저 로그인 기능을 담당합니다. **username**과 **password** 값을 클라이언트로부터 받습니다. 클라이언트 코드는 이렇습니다.

```tsx
const onSubmit = (values: z.infer<typeof RegisterUserSchema>) => {
  startTransition(() => {
    signInAction(values.username, values.password)
      .then((result) => {
        ...
      })
  })
}
```

그리고 이 기능의 핵심인 서버 엑션 코드입니다.


```ts
export async function signInAction(username: string, password: string) {

  const user = await signInUseCase(username, password)

  if (!user) {
    return {
      errors: "Invalid Credentials - Please Try Again"
    }
  }

  const session = await lucia.createSession(user.id, {})
  const sessionCookie = lucia.createSessionCookie(session.id)
  cookies().set(
    sessionCookie.name,
    sessionCookie.value,
    sessionCookie.attributes
  )

  return redirect("/dashboard")
}
```

하나하나 살펴봅시다. 클라이언트로부터 받은 값들은 `signInUseCase`이라는 함수로 전달됩니다. `signInUseCase` 함수를 살펴봅시다.

```ts
export async function signInUseCase(username: string, password: string) {

  const user = await db.query.userTable.findFirst({
    where: eq(userTable.username, username)
  })

  if (!user) {
    return null
  }

  const isPasswordCorrect = await verifyPassword(username, password)

  if (!isPasswordCorrect) {
    return null
  }

  return user
}
```

1. 전달받은 유저네임으로 유저가 데이터베이스에 존재하는지 확인합니다. 존재하지 않는다면 **null**값을 반환하여 클라이언트에 에러를 보냅니다
2. 유저네임이 데이터베이스에 존재한다면 비밀번호가 유효한지 확인합니다
3. 유효하지 않다면 **null**값을 반환합니다
4. 모든 체크가 통과하면 유저를 리턴합니다

`verifyPassword` 함수를 한 번 살펴봅시다.

```ts
async function verifyPassword(username: string, plainTextPassword: string) {
  const user = await db.query.userTable.findFirst({
    where: eq(userTable.username, username)
  })

  if (!user) {
    return false
  }

  const salt = user.salt
  const savedPassword = user.hashedPassword

  if (!salt || !savedPassword) {
    return false;
  }

  const hash = await hashPassword(plainTextPassword, salt)
  return user.hashedPassword == hash
}
```

유저네임과 비밀번호를 값으로 전달 받습니다. 로그인 과정을 조금 더 안전하게 만들기 위해, 해당 함수 내에서도 데이터베이스에 전달받은 유저네임이 존재하는지 확인합니다. 확인 후, 데이터베이스에 **salt**와 **password** 값들이 존재하는지 확인합니다. 존재한다면 해당 값들을 `hashPassword`함수에 전달합니다 (위에서 언급된 함수). 비밀번호가 유효하면 유저가 올바른 비밀번호를 입력했다는 것이고 체크가 통과됩니다.

그럼 다시 서버 엑션 코드로 돌아갑시다. 해당 부분을 살펴보죠.

```ts
const session = await lucia.createSession(user.id, {})
const sessionCookie = lucia.createSessionCookie(session.id)
cookies().set(
  sessionCookie.name,
  sessionCookie.value,
  sessionCookie.attributes
)
```

**세션**과 **세션 쿠키**를 생성해야 합니다. 해당 값들을 이용해 유저의 로그인 상태를 자동적으로 판단할 수 있기 때문입니다.

먼저, 해당 코드를 살펴봅시다.

```ts
const session = await lucia.createSession(user.id, {})
```

유저의 아이디를 **Lucia**의 [`createSession`](https://v2.lucia-auth.com/reference/lucia/interfaces/auth/#createsession) 함수로 전달합니다.

새로운 Lucia 인스턴스를 생성하며 세션 쿠키의 세부설정을 합니다.

```ts
const lucia = new Lucia(adapter, {
  sessionCookie: {
    expires: false,
    attributes: {
      secure: false
    }
  },
  sessionExpiresIn: new TimeSpan(20, "m"), // can set session expiration
  getUserAttributes: (attributes) => {
    return { ...attributes }
  }
});
```

**세션**을 만든 후, **세션 쿠키**를 생성합니다.

```ts
const sessionCookie = lucia.createSessionCookie(session.id)
```

이후, 이를 브라우저 쿠키로 저장합니다.

- `sessionCookie.name`: 쿠키 값의 이름을 설정합니다. 이 경우엔 **auth_session**로 설정됩니다
- `sessionCookie.value`: 쿠키 값을 설정합니다
- `sessionCookie.attributes`: 쿠키의 **httpOnly** 혹은 **maxAge**와 같은 설정을 합니다

쿠키까지 설정된 후, 유저를 대시보드 페이지로 이동시킵니다.

#### `sign-out-action`

해당 서버 액션은 유저를 로그아웃 시킵니다.

```ts
export async function signOutAction() {
  const { session } = await validateRequest();
  if (!session) {
    return {
      error: "Unauthorized"
    }
  }

  await lucia.invalidateSession(session.id)
  const sessionCookie = lucia.createBlankSessionCookie();
  cookies().set(
    sessionCookie.name,
    sessionCookie.value,
    sessionCookie.attributes
  )

  return redirect("/")
}
```

이후, `validateRequest`라는 함수로 세션의 유효 상태를 확인합니다. `validateRequest` 함수가 어떻게 작동하는지 확인합니다.

```ts
export const validateRequest = cache(
  async (): Promise<{ user: User; session: Session } | { user: null; session: null }> => {
    const sessionId = cookies().get(lucia.sessionCookieName)?.value ?? null;
    if (!sessionId) {
      return {
        user: null,
        session: null
      };
    }

    const result = await lucia.validateSession(sessionId);
    // next.js throws when you attempt to set cookie when rendering page
    try {
      if (result.session && result.session.fresh) {
        const sessionCookie = lucia.createSessionCookie(result.session.id);
        cookies().set(sessionCookie.name, sessionCookie.value, sessionCookie.attributes);
      }
      if (!result.session) {
        const sessionCookie = lucia.createBlankSessionCookie();
        cookies().set(sessionCookie.name, sessionCookie.value, sessionCookie.attributes);
      }
    } catch { ... }
    return result;
  }
);
```

해당 함수는 `cache`아 감싸여 있습니다. **sessionId** 가 존재하는지 확인합니다. 존재하지 않는다면 **user**와 **session**의 값으로 **null**을 반환합니다.

하지만 **sessionId**가 존재한다면 `validateSession`를 이용하여 세션을 확인합니다.

위에서 언급된 것 처럼 **sessionId**가 존재하지 않는다면 해당 코드에 걸려 **null**을 반환합니다.

```ts
if (!session) {
    return {
      error: "Unauthorized"
    }
  }
```

하지만 **sessionId**가 존재한다면 `invalidateSession`를 이용하여 세션을 invalidate합니다.

```ts
await lucia.invalidateSession(session.id)
const sessionCookie = lucia.createBlankSessionCookie();
cookies().set(
  sessionCookie.name,
  sessionCookie.value,
  sessionCookie.attributes
)

return redirect("/")
```

추가적으로 **blank session cookie**를 이용해 브라우저 쿠키를 없앱니다.

이후, 유저를 로그인 페이지로 이동시킵니다.

### 컨텐츠

- [get-posts-action](#get-posts-action)
- [get-post-action](#get-post-action)
- [create-post-action](#create-post-action)
- [update-post-action](#update-post-action)
- [delete-photo-action](#delete-photo-action)
- [delete-post-action](#delete-post-action)

#### `get-posts-action`

이름에서 알 수 있듯이 해당 서버 엑션은 대쉬보드 페이지에 표시될 게시물을 데이터베이스에서 쿼리합니다.

먼저, 유저가 로그인 했는지 확인합니다.

```ts
export async function getPostsAction(page: number, limit: number) {
  const { user } = await validateRequest()

  if (!user) {
    return {
      error: "ERROR - Not Authenticated"
    }
  }

  ...
}
```

이후, **allPosts** 와 **numberOfPosts**이라는 변수를 생성합니다.

Pagination을 위해 **offset** 값도 설정합니다.

```ts
const offset = (page - 1) * 11

allPosts = await db.query.posts.findMany({
  with: {
    photos: true
  },
  orderBy: [desc(posts.createdAt)],
  limit,
  offset // the number set here will tell drizzle orm to skip that amount of rows.
})
```

Pagination 관련해서는 [이 부분]()을 참고하시면 됩니다.

게시물을 쿼리한 후, drizzle orm을 활용해서 **numberOfPosts** 값도 반환합니다.

```ts
numberOfPosts = await db.select({ count: count() }).from(posts);
```

마지막으로 **allPosts** 와 **numberOfPosts** 값을 클라이언트로 반환합니다.

#### `get-post-action`

해당 서버 액션은 `/dashboard/post/[postId]` URL로 이동했을 때 특정한 게시물과 관련된 데이터를 쿼리합니다.

유저 로그인 상태를 확인 한 후, drizzle orm을 이용하여 게시물 데이터를 가져옵니다.

```ts
let post

try {
  post = await db.query.posts.findFirst({
    where: eq(posts.id, id),
    with: {
      photos: {
        orderBy: [asc(photos.createdAt)]
      }
    }
  })
} catch (error) {
  return {
    error: "Failed to retrieve post"
  }
}
```

복잡한 로직은 없습니다. **id** 값과 일치하는 게시물의 데이터를 쿼리합니다. **id**값을 클라이언트에서 가져옵니다. 또한, 게시물의 photo는 데이터베이스에 저장된 순서로 쿼리됩니다.

#### `create-post-action`

해당 서버 액션은 게시물을 하나 생성합니다. 해당 서버 액션의 로직을 구현하는 것이 이번 프로젝트를 진행하면서 가장 어려운 것 중 하나였습니다. AWS S3를 이용하는 것이 이 원인 중 하나였습니다.

먼저, 유저의 로그인 상태를 확인합니다.

이후, 클라이언트에서 받은 데이터를 각각 배열에 할당합니다.

```ts
export async function createPostAction(formData: FormData, postTitle: string) {

  // check user's authentication status

    const photoIdsArray = formData.getAll("photoId")
    const photoTitlesArray = formData.getAll("photoTitle")
    const photoDescriptionsArray = formData.getAll("photoDescription")
    const photoImagesArray = formData.getAll("photoImage")
    const photoCamerasArray = formData.getAll("camera")
    const photoLensArray = formData.getAll("lens")
    const photoFilmsArray = formData.getAll("film")

  } catch { ... }

...
}
```

클라이언트의 폼(form) 구조에 대해서는 [이 부분]()에서 설명합니다.

이후, **contentArray** 이라는 변수를 생성하여, 위 각각의 배열에 있는 값들을 하나의 오브젝트로 묶어줍니다.

```ts
let contentArray = []

for (let i = 0; i < photoIdsArray.length; i++) {
  contentArray[i] = {
    photoTitle: photoTitlesArray[i],
    photoDescription: photoDescriptionsArray[i],
    photoImage: photoImagesArray[i],
    camera: photoCamerasArray[i],
    lens: photoLensArray[i],
    film: photoFilmsArray[i]
  }
}
```

이 과정 이후, **contentArray** 값은 이렇게 생깁니다.

```ts
[
  {
    photoTitle: "",
    photoDescription: "",
    photoImage: "",
    camera: "",
    lens: "",
    film: ""
  },
  {
    photoTitle: "",
    photoDescription: "",
    photoImage: "",
    camera: "",
    lens: "",
    film: ""
  }
  ...
]
```

다음으로, drizzle을 사용해서 새로운 게시물(post)을 생성합니다.

```ts
const post = await db.insert(posts).values({
  title: postTitle
}).returning({
  postId: posts.id,
})
```

**postId** 값도 이후에 사용해야 하기 때문에 반환해줍니다.

이후, 실제 사진들(+ 각 사진의 설명, 제목, 이미지 주소 등)을 저장해줍니다. 먼저, 이 코드를 한 번 살펴봅시다.

```ts
const uploadAllImages = async (contents: any) => {
  const client = new S3Client({
    region: process.env.BUCKET_REGION,
    credentials: {
      accessKeyId: process.env.ACCESS_KEY_ID as string,
      secretAccessKey: process.env.SECRET_ACCESS_KEY as string,
    }
  })
  for (let i = 0; i < contents.length; i++) {
    await createPhoto(contents[i], client)
  }
}

await uploadAllImages(contentArray)
```

`uploadAllImages`이라는 함수를 생성합니다. 이 함수는 제 AWS S3 버킷에 엑세스하기 위해 필요합니다. **contentArray**의 length번 루핑하여 루핑할 때마다 `createPhoto`함수를 호출합니다. 그럼, `createPhoto`함수를 살펴봅시다.

```ts
const createPhoto = async (content, client) => {

  try {
    const imageName = randomBytes(16).toString("hex")

    const command = new PutObjectCommand({
      Bucket: process.env.BUCKET_NAME,
      Key: imageName,
      ContentType: content.photoImage.type
    })


    const preSignedUrl = await getSignedUrl(client, command, {
      expiresIn: 30
    })

    const upload = await fetch(preSignedUrl, {
      method: "PUT",
      body: content.photoImage,
      headers: {
        "Content-Type": content.photoImage.type
      }
    })
  } catch { ... }
...
}
```

루프를 할 때마다 사진이 S3에  업로드 + 이미지 주소 반환 + drizzle을 활용항 데이터베이스에 저장의 과정을 거칩니다.

먼저, `crypto`의 **randomBytes**를 활용하여 사진의 고유 이름을 생성해줍니다. 이후, **PutObjectCommand**의 인스턴스를 하나 새로 생성하여 필요한 설정을 해줍니다.

**signed URL**을 얻게되는데, 해당 URL은 저의 S3버킷에 엑세스 하는데 필요합니다. **signed URL**을 받은 후, 해당 주소로 사진을 저장합니다.

```ts
if (upload.ok) {
  const s3FileUrl = `https://${process.env.CDN_URL}.cloudfront.net/${imageName}`

  await db.insert(photos).values({
    title: content.photoTitle,
    description: content.photoDescription,
    imageUrl: s3FileUrl,
    camera: content.camera,
    lens: content.lens,
    film: content.film,
    postId: post[0].postId
  })

} else {
  return {
    error: "Error while creating photo",
  }
} else { ... }
```

S3로 업로드가 성공적이면 **s3FileUrl**라는 변수를 생성하여 사진의 주소를 값으로 할당합니다. 해당 주소를 사용하여 유저에게 사진을 보여줄 겁니다. 또한, AWS Cloudfront를 사용하기 때문에, 사진의 주소를 이렇게 설정하였습니다.

```
https://${process.env.CDN_URL}.cloudfront.net/${imageName}
```

사진의 주소를 반환받은 후, 사진과 관련된 정보(사진의 제목, 설명, 장비 내용 등)를 저장하게 됩니다. 위에서 언급된 **postId**가 중요한데요, 이것은 게시물(post)과 사진(photo) 사이에 one-to-many relationship을 설정하기 위함입니다.

최종적으로, 대쉬보드 페이지가 revalidate되며 **postTitle**와 **isSuccess**값을 클라이언트로 반환합니다.

```ts
revalidatePath("/dashboard")

return {
  data: postTitle,
  isSuccess: true
}
```

#### `update-post-action`

해당 서버 액션은 기존 게시물의 내용을 수정하는 데에 사용됩니다.

- 게시물의 제목을 수정합니다
- 새로운 사진을 추가하거나 기존 사진을 삭제합니다
- 이미 존재하는 사진 + 사진의 설명/장비 등을 수정합니다

유저의 로그인 상태를 확인한 후, 이미 존재하는 게시물의 데이터를 각각의 어레이에 할당합니다.

```ts
const photoIdsArray = formData.getAll("photoId")
const photoTitlesArray = formData.getAll("photoTitle")
const photoDescriptionsArray = formData.getAll("photoDescription")
const photoImagesArray = formData.getAll("photoImageUrl")
const photoNewImagesArray = formData.getAll("newPhotoImage")
const photoCamerasArray = formData.getAll("photoCamera")
const photoLensArray = formData.getAll("photoLens")
const photoFilmsArray = formData.getAll("photoFilm")
const photoPostIdsArray = formData.getAll("photoPostId")
```

