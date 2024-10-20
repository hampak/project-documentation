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

