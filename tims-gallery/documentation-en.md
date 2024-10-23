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

- [Admin Page](#admin-page)
- [Gallery Page](#gallery-page)

## Admin Page
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
