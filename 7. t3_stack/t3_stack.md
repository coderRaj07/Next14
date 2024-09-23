# **T3 Stack with tRPC, TanStack Query, and Bearer Token Authentication**

This project demonstrates how to set up a Next.js application using the T3 stack with **tRPC** for API communication, **TanStack Query** for data fetching, and handling Bearer token authentication.

## **Table of Contents**

1. [Project Structure](#project-structure)
2. [Setup Guide](#setup-guide)
3. [API Helpers](#api-helpers)
4. [Creating Routers](#creating-routers)
   - [User Router](#user-router)
   - [Post Router](#post-router)
   - [Comments Router](#comments-router)
5. [tRPC API Handler](#trpc-api-handler)
6. [TanStack Query Integration](#tanstack-query-integration)
7. [Example Components](#example-components)
   - [Fetch User Data](#fetch-user-data-with-token)
   - [Fetch Filtered Posts](#fetch-filtered-posts)
   - [Fetch Users with Query Parameters](#fetch-users-with-query-parameters)
8. [Running the Project](#running-the-project)

---

## **Project Structure**

```
/src
  ├── /app
  │   └── /api
  │       └── /trpc
  │           └── route.ts         // Central tRPC API handler with context for Bearer token
  ├── /routers
  │   ├── users.ts                 // User router with token-based and no-token routes
  │   ├── posts.ts                 // Post router with token-based and no-token routes
  │   └── comments.ts              // Comments router with token-based and no-token routes
  ├── /utils
  │   ├── trpc.ts                  // tRPC client setup
  │   └── api.ts                   // Helper functions for making authenticated API calls
  └── /pages
      └── _app.tsx                 // tRPC provider setup
      └── /components
          ├── UserPage.tsx         // Example of fetching user data with token
          ├── FilteredPostPage.tsx  // Example of fetching posts with dynamic query params
          ├── UserList.tsx          // Example of fetching users with query parameters
          └── PostPage.tsx         // Example of fetching post data without token
```

---

## **Setup Guide**

### **1. Install Dependencies**

Install the necessary packages for tRPC, Next.js, TanStack Query, and TypeScript:

```bash
npm install next react react-dom @trpc/server @trpc/client @trpc/react @trpc/next zod tanstack/react-query
```

### **2. API Helpers**

In `/src/utils/api.ts`, create helper functions to handle **GET** and **POST** requests to 3rd-party APIs, accommodating **Bearer token authentication**.

```ts
// src/utils/api.ts
export async function fetchAPI(url: string, token: string) {
  const res = await fetch(url, {
    headers: {
      'Authorization': `Bearer ${token}`,
    },
  });
  if (!res.ok) {
    throw new Error('Error fetching data');
  }
  return res.json();
}

export async function postAPI(url: string, body: any, token: string) {
  const res = await fetch(url, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  });
  if (!res.ok) {
    throw new Error('Error posting data');
  }
  return res.json();
}

export async function fetchAPIWithParams(url: string, params: Record<string, string>, token: string = '') {
  const queryString = new URLSearchParams(params).toString();
  const fullUrl = `${url}?${queryString}`;
  
  const res = await fetch(fullUrl, {
    headers: {
      'Authorization': token ? `Bearer ${token}` : undefined,
    },
  });
  
  if (!res.ok) {
    throw new Error('Error fetching data');
  }
  
  return res.json();
}
```

---

## **Creating Routers**

Each router handles different services (e.g., users, posts, comments) with varying authentication requirements.

### **User Router**

```ts
// src/routers/users.ts
import * as trpc from '@trpc/server';
import { z } from 'zod';
import { fetchAPI, postAPI, fetchAPIWithParams } from '@/utils/api';

const apiBaseUrl = 'https://api.example.com/v1';

export const usersRouter = trpc.router()
  .query('getUser', {
    input: z.object({ userId: z.string() }),
    async resolve({ input, ctx }) {
      const { token } = ctx;  // Using token for this route
      return await fetchAPI(`${apiBaseUrl}/users/${input.userId}`, token);
    },
  })
  .query('listUsers', {
    input: z.object({
      list: z.string().optional(),    // Accept optional 'list' parameter
      limit: z.number().optional(),    // Accept optional 'limit' parameter
      pageIndex: z.number().optional(), // Accept optional 'pageIndex' parameter
    }),
    async resolve({ input }) {
      const { list, limit, pageIndex } = input;
      const params: Record<string, string> = {};
      
      if (list) params.list = list;             // Add to params if provided
      if (limit) params.limit = limit.toString(); // Convert limit to string
      if (pageIndex) params.pageIndex = pageIndex.toString(); // Convert pageIndex to string

      const queryString = new URLSearchParams(params).toString(); // Build query string
      return await fetchAPI(`${apiBaseUrl}/users?${queryString}`, ''); // Fetch without token
    },
  })
  .mutation('createUser', {
    input: z.object({
      name: z.string(),
      email: z.string(),
    }),
    async resolve({ input, ctx }) {
      const { token } = ctx;
      return await postAPI(`${apiBaseUrl}/users`, input, token);
    },
  });
```

### **Post Router**

```ts
// src/routers/posts.ts
import * as trpc from '@trpc/server';
import { z } from 'zod';
import { fetchAPI, postAPI, fetchAPIWithParams } from '@/utils/api';

const apiBaseUrl = 'https://api.example.com/v1';

export const postsRouter = trpc.router()
  .query('getPost', {
    input: z.object({ postId: z.string() }),
    async resolve({ input, ctx }) {
      const { token } = ctx;  // Requires token
      return await fetchAPI(`${apiBaseUrl}/posts/${input.postId}`, token);
    },
  })
  .query('listPosts', {
    async resolve() {
      // No token required
      return await fetchAPI(`${apiBaseUrl}/posts`, '');
    },
  })
  .mutation('createPost', {
    input: z.object({
      title: z.string(),
      content: z.string(),
    }),
    async resolve({ input, ctx }) {
      const { token } = ctx;
      return await postAPI(`${apiBaseUrl}/posts`, input, token);
    },
  })
  .query('getPostsWithFilters', {
    input: z.object({
      category: z.string().optional(),
      author: z.string().optional(),
      date: z.string().optional(),
    }),
    async resolve({ input, ctx }) {
      const { token } = ctx;
      const params = {
        ...(input.category && { category: input.category }),
        ...(input.author && { author: input.author }),
        ...(input.date && { date: input.date }),
      };
      
      return await fetchAPIWithParams(`${apiBaseUrl}/posts`, params, token);
    },
  });
```

### **Comments Router**

```ts
// src/routers/comments.ts
import * as trpc from '@trpc/server';
import { z } from 'zod';
import { fetchAPI, postAPI } from '@/utils/api';

const apiBaseUrl = 'https://api.example.com/v1';

export const commentsRouter = trpc.router()
  .query('getComment', {
    input: z.object({ commentId: z.string() }),
    async resolve({ input, ctx }) {
      const { token } = ctx;  // Requires token
      return await fetchAPI(`${apiBaseUrl}/comments/${input.commentId}`, token);
    },
  })
  .query('listComments', {
    async resolve() {
      // No token required
      return await fetchAPI(`${apiBaseUrl}/comments`, '');
    },
  })
  .mutation('createComment', {
    input: z.object({
      postId: z.string(),
      content: z.string(),
    }),
    async resolve({ input, ctx }) {
      const { token } = ctx;
      return await postAPI(`${apiBaseUrl}/comments`, input, token);
    },
  });
```

---

## **tRPC API Handler**

In `/src/app/api/trpc/route.ts`, create a central handler for the tRPC context and merge the routers.

```ts
// src/app/api/trpc/route.ts
import * as trpc from '@trpc/server';
import { usersRouter } from '@/routers

/users';
import { postsRouter } from '@/routers/posts';
import { commentsRouter } from '@/routers/comments';

export const trpcRouter = trpc.router()
  .merge('users.', usersRouter)
  .merge('posts.', postsRouter)
  .merge('comments.', commentsRouter);

export type AppRouter = typeof trpcRouter;

export const createContext = ({ req, res }) => {
  const token = req.headers.authorization?.split(' ')[1] || ''; // Extract Bearer token
  return { token };
};
```

---

## **TanStack Query Integration**

In your `_app.tsx` file, set up TanStack Query and tRPC provider.

```tsx
// src/pages/_app.tsx
import { AppProps } from 'next/app';
import { trpc } from '@/utils/trpc';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient();

const MyApp = ({ Component, pageProps }: AppProps) => {
  return (
    <QueryClientProvider client={queryClient}>
      <Component {...pageProps} />
    </QueryClientProvider>
  );
};

export default trpc.withTRPC(MyApp);
```

---

## **Example Components**

### **Fetch User Data with Token**

```tsx
// src/components/UserPage.tsx
import { trpc } from '@/utils/trpc';

const UserPage = () => {
  const userId = '1'; // Example user ID
  const { data, isLoading } = trpc.users.getUser.useQuery({ userId });

  return (
    <div>
      <h1>User Details</h1>
      {isLoading ? (
        <p>Loading...</p>
      ) : (
        <p>Name: {data.name}</p>
      )}
    </div>
  );
};

export default UserPage;
```

### **Fetch Filtered Posts**

```tsx
// src/components/FilteredPostPage.tsx
import { trpc } from '@/utils/trpc';
import { useState } from 'react';

const FilteredPostPage = () => {
  const [category, setCategory] = useState('tech');
  const [author, setAuthor] = useState('john');
  const [date, setDate] = useState('2024-09-23');

  const { data, isLoading } = trpc.posts.getPostsWithFilters.useQuery({
    category,
    author,
    date,
  });

  return (
    <div>
      <h1>Filtered Posts</h1>
      {isLoading ? (
        <p>Loading posts...</p>
      ) : (
        <div>
          {data?.map((post: any) => (
            <p key={post.id}>{post.title}</p>
          ))}
        </div>
      )}
    </div>
  );
};

export default FilteredPostPage;
```

### **Fetch Users with Query Parameters**

```tsx
// src/components/UserList.tsx
import { trpc } from '@/utils/trpc';
import { useState } from 'react';

const UserList = () => {
  const [list, setList] = useState('admin');
  const [limit, setLimit] = useState(5);
  const [pageIndex, setPageIndex] = useState(5);

  const { data, isLoading } = trpc.users.listUsers.useQuery({
    list,
    limit,
    pageIndex,
  });

  return (
    <div>
      <h1>User List</h1>
      {isLoading ? (
        <p>Loading users...</p>
      ) : (
        <div>
          {data?.map((user: any) => (
            <p key={user.id}>{user.name}</p>
          ))}
        </div>
      )}
    </div>
  );
};

export default UserList;
```

---

## **Running the Project**

To run the project, ensure you have the necessary dependencies installed and execute:

```bash
npm run dev
```

This will start your Next.js application, allowing you to interact with the tRPC APIs and test your components. 
