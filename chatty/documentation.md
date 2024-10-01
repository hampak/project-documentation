# Table of Contents
1. [Project Overview](#project-overview)
2. [API Documentation](#api-documentation)
3. Features
4. Notable Bugs & Problems I encountered
5. Learning Experience

# Project Overview
Chatty is a chatting application where users can communicate with each other. The idea of real-time communication has always intrigued me and I was really interested in building a project like this. I learned a lot of technologies while building this project such as **redis** and **web sockets**.

I built this project using the tech stack / packages below:
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

I chose **ExpressJS** because I wanted to be more comfortable using it. For the main database, I used **MongoDB** and for data requiring a lot of read/writes, I used **redis(upstash)**.

One important thing to note here is that I didn't use an authentication service such as Clerk or Kinde. I wanted to learn **OAuth 2.0** so I didn't use any of those third-party authentication service providers.

In the frontend, I used **React**. For routing and navigation I used **react router dom**. For data fetching, I used **tanstack query**. For styling I used **tailwindcss**.

Through this project, I wanted to fortify my knowledge on key concepts of some of the most important technologies used around the world.

# API Documentation
