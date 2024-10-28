# Table of Contents
1. [Project Overview](#project-overview)
2. [API Documentation](#api-documentation)
4. [Features](#features)
5. [Notable Bugs & Problems I encountered](#notable-bugs--problems-i-encountered)
6. [Learning Experience](#learning-experience)

# Project Overview
Tim's Gallery is a photo gallery where I upload the photos I take. I love photography, whether it be film or digital. Building a photo gallery was one of the things I wanted to do the most and I finally did it.

I built both an admin page (where only I can access and upload photos/posts) and the actual gallery page. THis documentation will be about both.

For the admin page, I used the following tech stack:

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

I used **NextJS** because of the simplicity it provides. I used **aws s3** for storing the photos. For data fetching on the client, I used **tanstack query** and used **zod** for validation. I used **drizzle orm** to simplify data reads and writes from my database. I used **lucia** for authentication (although the maintainer says that it will be [deprecated](https://github.com/lucia-auth/lucia/discussions/1714) in March 2025 so I might try implementing auth in a different way). For styling I used **tailwindcss**. I used **blocknote**, which was a block-based text editor. I love this library as it gives me a lot of flexibility of implementing a block-like editor (similar to that of Notion).

One of the most difficult things I faced when building this project was using the **react hook form** library. As the main feature of this gallery was to upload posts with a lot of fields (such as the photo itself, post title, photo title, description and etc), implementing the forms part was quite complicated. However, I found a way in the end which I have explained in the later parts of this documentation.

For the gallery page, I used the following tech stack:

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

It's pretty similar when comparing to the admin page.

# API Documentation

The API documentation section is divided into the admin page and the gallery page. Feel free to click on the parts you'd like to see in the list below.

- [Admin Page API](#admin-page-api)
- [Gallery Page API](#gallery-page-api)

## Admin Page API
- [authentication](#authentication)
- [content](#content)

### Authentication
- [sign-up-action](#sign-up-action)
- [sign-in-action](#sign-in-action)
- [sign-out-action](#sign-out-action)

#### `sign-up-action`
Although the sole user of the admin page is me, I still felt that I needed to document how I implemented the user registration feature. It felt weird building this feature because I was the only user. However, I wanted to learn how to implement user registration feature using **lucia auth** with drizzle orm.

Now, I chose to use **drizzle orm** and **lucia** as the tech stack for authentication to challenge myself. At the point of developing this feature, there weren't a lot of sources that explained how to do this. In the end, I found a way to do this.

First, the client sends the required fields to register a user. For now, I wanted to make it simple so it was just the **username** and the **password**.

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

This is sent to the server action.

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

Within the server action, the values are passed in a function called `registerUserUseCase`. Here's the logic I implemented in this function:

1. First, we check to see if a user with the username already exists in the database. If it does, we return an error
2. If we past the check above, we then move onto the next part which is generating the required **salt** and **hash** to store in the database. These values will be used in the authentication process
3. We then insert a new user in our database

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

We check if the username already exists.

Then we generate the **salt** and **hash**.

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

We generate a salt using the **crypto** module of NodeJS. After that, we create a **hashed password**. We do this by passing in the **password** and the **salt** to a function called `hashPassword`. This function creates a hashed password with the logic below:

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

After creating both the **salt** and **hash**, we insert it in our postgres database.

Lastly, we return the **user** and redirect the user to the main page.

#### `sign-in-action`

This server action signs in the user (me). The server action function takes in an **username** and **password** from the client.

Here's the client-side code:

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

And here's the server action function:

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

Let's go over it step by step. The values received from the client is then passed on to a function called `signInUseCase`. Here is the `signInUseCase` function:

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

1. The code first checks to see if a user with the username exists in the database. If it doesn't exist, we return null which will send an error to the client code
2. If a user with the username does exist, we then check to see if the password is valid. If it's not a valid password, we return null which will send an error to the client code
3. If all checks pass, we return the user

Let's take a look at the `verifyPassword` function.

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

As we can see here, the function takes in the **username** and the **password**. Just to be sure, we check once more if a user with the **username** exists in the database. After the check, we check to see if the **salt** and **password** exists in the database. If it does, we put the **salt** and the **password(from the client)** in the `hashPassword` function which was mentioned above. If the password is valid, we know that the user inputted the correct password.

When then move to this piece of code in our server action:

```ts
const session = await lucia.createSession(user.id, {})
const sessionCookie = lucia.createSessionCookie(session.id)
cookies().set(
  sessionCookie.name,
  sessionCookie.value,
  sessionCookie.attributes
)
```

We want to create a session and a session cookie. These will be used to automatically validate the already-authenticated user.

First, let's take a look at thie line of code:

```ts
const session = await lucia.createSession(user.id, {})
```

We send the authenticated user's id to Lucia's [`createSession`](https://v2.lucia-auth.com/reference/lucia/interfaces/auth/#createsession) function.

In the code, it creates a new Lucia instance where I can set configurations related to session and session cookies.

For example, like this:

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

After creating the session, we create the **session cookie**.

```ts
const sessionCookie = lucia.createSessionCookie(session.id)
```

After this, we set it as cookies.

- `sessionCookie.name`: sets the name of the cookie. In this case, it's **auth_session**
- `sessionCookie.value`: sets the value of the session cookie.
- `sessionCookie.attributes`: sets the options of the session cookie such as the **httpOnly** or **maxAge** configurations

After this, the user is redirected to the dashboard page.

#### `sign-out-action`

This server logs out the current user.

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

It first checks the session status by using the `validateRequest` function. Let's take a look at how this function works.


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

The function is wrapped in `cache`. It does some checks to see if the **sessionId** exists. If it doesn't, it will return **null** values for both the **user** and the **session**.

However, if there is a **sessionId**, we validate the session using `validateSession`. After some checks, we return the result.

Back to the code above, if there isn't a **sessionId**, the code returns null which will be caught in this line here:

```ts
if (!session) {
    return {
      error: "Unauthorized"
    }
  }
```

However, if there is a **sessionId**, we invalidate the session using the `invalidateSession`.

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

We also create a **blank session cookie** which we will set as the browser cookie.

Then, we navigate the user back to the login page.

### Content

- [get-posts-action](#get-posts-action)
- [get-post-action](#get-post-action)
- [create-post-action](#create-post-action)
- [update-post-action](#update-post-action)
- [delete-photo-action](#delete-photo-action)
- [delete-post-action](#delete-post-action)

#### `get-posts-action`

As the name suggests, this server action fetches the list of posts which will be shown in the dashboard page.

First, we check to see that the user is authenticated.

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

Next, we create two variables: **allPosts** and **numberOfPosts**.

We set the **offset** value which will be vital for the pagination feature.

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

Pagination will be discussed more in-depth in [**this**]() part of the post.

After fetching the posts, we also return the value **numberOfPosts** with drizzle orm.

```ts
numberOfPosts = await db.select({ count: count() }).from(posts);
```

Finally, we return the **allPosts** and **numberOfPosts** to the client.

#### `get-post-action`

This server action is used to fetch the specific post when we navigate to a `/dashboard/post/[postId]` page.

After checking the authentication status, it fetches data using drizzle orm.

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

Nothing complicated here. We retrieve the post that matches with the **id** value. The **id** is passed from the client. We also order the photos in the order in which they were created.

We return the post to the client.

#### `create-post-action`

This server action creates a post. This server action was one of the most difficult parts I faced while building this project. Mainly because it involved using AWS S3 to store the images.

First, we check the authentication status of the user.

Next, we store the list of data we received from the client in their separate arrays.

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

I've written a detailed explanation of the complex form structure in [**this**]() part of the documentation.

We then create an array called **contentArray** where we will loop over each of the arrays listed above and store them in objects.

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

The **contentArray** array will looks something like this:

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

After this, we create a new post.

```ts
const post = await db.insert(posts).values({
  title: postTitle
}).returning({
  postId: posts.id,
})
```

We also return the **postId** value because we'll use it later on.

We then move on to storing the actual photos (along with their description, title, image url etc). Let's take a look at this code here:

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

Here, I've created a function called `uploadAllImages`. This is an async function and it includes the code needed to access my AWS S3 bucket. After that, the code loops over the length of the contents, and each time the function `createPhoto` is called. Let's take a look at the `createPhoto` function.

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

I coded a logic so that on every loop, a photo is uploaded to S3 + retrieve the image url + store data using drizzle.

First, the code creates a name for the image using **randomBytes** from `crypto`. Then, we create a new instance of the **PutObjectCommand** and fill in the configurations.

We get a **signed URL** which we will use to upload it to my S3 bucket. We then add the image to the S3 bucket.

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

If the upload is successful, we set a variable **s3FileUrl**. This will be the actual **object url** that will be stored in the database. Because I'm using AWS Cloudfront, I've set the object url to look like this:

```
https://${process.env.CDN_URL}.cloudfront.net/${imageName}
```

After that, the code adds a new photo with the corresponding values. The **postId** value is important here as it creates a **one-to-many** relationship between the post and photo(s).

Finally, the dashboard path is revalidated and the **postTitle** data along with the **isSuccess** value is passed on to the client.

```ts
revalidatePath("/dashboard")

return {
  data: postTitle,
  isSuccess: true
}
```

#### `update-post-action`

This server action updates the content of a post. It can

- Update the **title** of a post
- Add or delete photos
- Edit already-existing photos
- Delete the post entirely

After checking the authentication status, we first store the data of the already existing photo(s) in their own arrays.

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

What's noticeably different is the fact that there is a new array: **photoNewImagesArray**

This is to store the new image. For example, let's say that I uploaded the wrong photo and saved it in my database. Then, I would reupload another image and that image will be stored in this array.

After this, we store the **newly created photos** in a whole different set of arrays.

```ts
const newPhotoIdsArray = formData.getAll("newPhotoId")
const newPhotoTitlesArray = formData.getAll("newPhotoTitle")
const newPhotoDescriptionsArray = formData.getAll("newPhotoDescription")
const newPhotoImageArray = formData.getAll("photoImage")
const newCameraArray = formData.getAll("newCamera")
const newFilmArray = formData.getAll("newFilm")
const newLensArray = formData.getAll("newLens")
```

We then create two arrays which will store and organize the data like what I did above.

```ts
let existingContentArray = []
let newContentArray = []

for (let i = 0; i < photoIdsArray.length; i++) {
  existingContentArray[i] = {
    photoId: photoIdsArray[i],
    photoTitle: photoTitlesArray[i],
    photoDescription: photoDescriptionsArray[i],
    photoImage: photoImagesArray[i],
    newPhotoImage: photoNewImagesArray[i],
    camera: photoCamerasArray[i],
    lens: photoLensArray[i],
    film: photoFilmsArray[i]
  }
}

for (let i = 0; i < newPhotoIdsArray.length; i++) {
  newContentArray[i] = {
    newPhotoTitle: newPhotoTitlesArray[i],
    newPhotoDescription: newPhotoDescriptionsArray[i],
    photoImage: newPhotoImageArray[i],
    newCamera: newCameraArray[i],
    newFilm: newFilmArray[i],
    newLens: newLensArray[i]
  }
}
```

After this, we update the title of the post.

```ts
await db.update(posts)
  .set({
    title: postTitle
  })
  .where(eq(posts.id, postId))
```

Then, it's the same logic as how I implemented the `create-post-action` server action.

```ts
const editAllImages = async (existingContents: any) => {
  const client = new S3Client({
    region: process.env.BUCKET_REGION,
    credentials: {
      accessKeyId: process.env.ACCESS_KEY_ID as string,
      secretAccessKey: process.env.SECRET_ACCESS_KEY as string,
    }
  })
  for (let i = 0; i < existingContents.length; i++) {
    await updatePhoto(existingContents[i], client)
  }
}

const createAllImages = async (newContents: any) => {
  const client = new S3Client({
    region: process.env.BUCKET_REGION,
    credentials: {
      accessKeyId: process.env.ACCESS_KEY_ID as string,
      secretAccessKey: process.env.SECRET_ACCESS_KEY as string,
    }
  })
  for (let i = 0; i < newContents.length; i++) {
    await createPhoto(newContents[i], client)
  }
}

await editAllImages(existingContentArray)
await createAllImages(newContentArray)
```

I created separate functions `editAllImages` and `createAllImages`.

- `editAllImages` - The function which will **update** the already existing photos using the `updatePhoto` function
- `createAllImages` - The function which will **create** new photos and add them to this post using the `createPhoto` function

Let's take a look at the `updatePhoto` function:

```ts
const updatePhoto = async (existingContents, client) => {

  if (existingContents.newPhotoImage === "undefined") {
    
    await db.update(photos)
      .set({
        title: existingContents.photoTitle,
        description: existingContents.photoDescription,
        camera: existingContents.camera,
        film: existingContents.film,
        lens: existingContents.lens,
        postId: existingContents.postId,
      })
      .where(eq(photos.id, existingContents.photoId))
  } else { ... }
```

Every time we loop over the **existingContents** array, we check to see if there is a value in **newPhotoImage**. If there isn't one, I've set the value to be "undefined". If this is the case, it means that the user hasn't changed the photo from the client. This means that there's no S3 involved and all we have to do is update the fields **excluding** the **imageUrl**.

However, what if the user changes the image? In that case, we move on to the **else** condition and go through the same logic mentioned [above](#create-post-action).

> You might be wondering: "Isn't it inefficient to update the fields even if they weren't changed?". That is absolutely correct. However, I had my reasons for implementing the logic like this which I have explained [here](#) in the documentation.

For the `createPhoto` function, it's the same logic for mentioned [above](#create-post-action).

#### `delete-photo-action`

This is a simple server action that deletes a photo. After checking the authentication status, it takes in the **photoId** value where we use drizzle orm to delete it rom our database.

```ts
try {
  await db.delete(photos)
    .where(eq(photos.id, photoId))
} catch (error) {
  return {
    error: "Error while deleting photo"
  }
}
```

> Deleting the image object in my S3 bucket hasn't been implemented yet

### `delete-post-action`

This server action deletes the entire post. After checking the authentication status, it takes in the **postId** and deletes it from my database.

```ts
try {
  await db.delete(posts)
    .where(eq(posts.id, postId))
} catch (err) {
  return {
    error: "Error while deleting post"
  }
}
```

> Deleting the image object(s) in my S3 bucket hasn't been implemented yet

## Gallery Page API
- [posts](#posts)

### Posts
- [get-infinite-posts-action](#get-infinite-posts-action)
- [get-specific-post-action](#get-specific-post-action)

#### `get-infinite-posts-action`

This is the server action that gets called in the main page of the gallery.

It uses the `useInfiniteQuery` function from **tanstack query**. Here is the server action code:

```ts
let postsSegment

try {
  postsSegment = await db.query.posts.findMany({
    with: {
      photos: true
    },
    where: cursor ? lt(posts.cursor, cursor) : undefined,
    limit: 9,
    orderBy: [desc(posts.createdAt)]
  })
} catch (error) {
  return {
    error: "Failed to retrieve posts :("
  }
}
```

I've made it so that it fetches 9 posts at a time. THe infinite query feature is explained more in detailed in [**this**]() part of the documentation.

#### `get-specific-post-action`

This server action fetches the data of a specific post. It gets called when a user clicks on a post.

```ts
try {
  post = await db.query.posts.findFirst({
    where: eq(posts.id, postId),
    with: {
      photos: {
        orderBy: [asc(photos.createdAt)]
      }
    },
    orderBy: [desc(posts.createdAt)]
  })

} catch (error) {
  return {
    error: "Failed to retrieve posts :("
  }
}
```

I used drizzle orm to fetch the specific post. The photos are sorted and returned in the order they were created in the admin page.

# Features

This section will talk about the features of Tim's Gallery. I've divided this section into 2 parts: the admin page and the gallery page.

- [Admin Page Feature](#admin-page-feature)
- [Gallery Page Feature](#gallery-page-feature)

## Admin Page Feature

- [Creating a Post with a Single Photo](#creating-a-post-with-a-single-photo)
- [Creating a Post with Multiple Photos](#creating-a-post-with-multiple-photos)
- [Editing an Existing Photo in a Post](#editing-an-existing-photo-in-a-post)
- [Adding a New Photo in a Post](#adding-a-new-photo-in-a-post)
- [Deleting a Photo in a Post](#deleting-a-photo-in-a-post)
- [Deleting a Post](#deleting-a-post)

#### `Creating a Post with a Single Photo`

https://github.com/user-attachments/assets/d9fcef21-be85-4d06-a9f7-29cfa81c6990

#### `Creating a Post with Multiple Photos`

#### `Editing an Existing Photo in a Post`

https://github.com/user-attachments/assets/1291f0f9-2e98-4ba4-a763-e3c100d3d28f

#### `Adding a New Photo in a Post`

https://github.com/user-attachments/assets/ffa670d5-471b-4e08-9fe4-8b5c4652fc14

#### `Deleting a Photo in a Post`

https://github.com/user-attachments/assets/4c247b07-7936-451d-a750-53e64579c1bf

#### `Deleting a Post`

https://github.com/user-attachments/assets/bfd61865-8fdf-45e6-88fc-1716c8aa9367

## Gallery Page Feature

- [Parellel Routes and Route Interception]()
- [Infinite Querying]()
- [Creating posts with React Hook Form]()
