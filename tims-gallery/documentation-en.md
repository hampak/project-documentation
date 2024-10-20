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

