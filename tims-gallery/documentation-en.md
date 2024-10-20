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
