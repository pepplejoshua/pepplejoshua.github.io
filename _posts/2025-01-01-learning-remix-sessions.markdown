---
layout: post
title:  "learning remix: tracking user preferences using session cookies"
date:   2025-01-01 02:03:00 -0700
categories: programming ts webdev
---

# learning webdev: tracking user preferences using session cookies in remix.

Starting a series of short notes on things I learn doing webdev. I have been trying [Remix](https://remix.run/) for a little
while to write web applications. Prior to using Remix, my only experience working in a large webdev codebase (professionally
or otherwise) came when I worked at TracxTMS (a trucking software company). We used Php / Laravel with MySQL, which was a
lot of fun since Laravel is a delightful walled-garden with a lot of the tool you need to thrive. Although the frontend bindings
to frameworks like React are not the best to work with (from my experience) although I have seen some projects already working
on solving that.

Back to Remix. I have been in a couple of situations where I would like to track some user preferences that affects how the Ui
is rendered (does the user want dark mode or light mode or did they previously have their sidebar open). When I ran into the dark
/ light mode problem initially, I solved it (very unsatisfactorily, might I add) using code in the `root.tsx` of my app that might
have looked something like:

```tsx
import { useLoaderData } from "@remix-run/react";

export const loader = async () => {
  // Fetch user's theme preference or use a default
  const theme = "light"; // Replace with actual logic to fetch theme
  return { theme };
};

export default function App() {
  const { theme } = useLoaderData();

  return (
    <html lang="en">
      <head>
        {/* other items... */}
        <script
          {% raw %}dangerouslySetInnerHTML={{
            __html: `
              (function() {
                // some code to set dark mode or something of the sort
              })();
            `,
          }}{% endraw %}
        />
      </head>
      <body>
        {/*  content */}
      </body>
    </html>
  );
}
```

It did not work very well and more often than not, I would see the flicker from light mode to dark mode or vice-versa as
it reconciled server side rendered pages with client-side state. It was not very clean, so I abandoned it and removed it
from the app.

Sometime last year, OpenAi moved the ChatGPT website to Remix. The Ui for ChatGPT provides a collapsible sidebar AND
light/ mode. When the page was refreshed, it rendered perfectly with 0 flickers from the sidebar and 0 color changes as the
theme was already known. I saw this a couple of day ago and to me this meant it was perfectly solvable and I should learn how
to solve it.

My second attempt was no better than the first, since I attempted to combine the localStorage on the user's browser with
an additional db field on the User model in the db. This did not work very well, since
1. I cannot access the user's localStorage when I actually perform the server side rendering of the page, and
2. Even though the db would get the fresh value, my authentication system did not read the db each time the page was refreshed
(for the user information which was just updated), since it already inserted a session cookie on login. This meant that the db
value was really not very useful or even accessed. That might have been a performance nightmare eventually as well.

After some thinking and prompting Claude Sonnet 3.5, I realized I was not making good use of the tools available to me
since I could just update the session cookie (and not the db) I set in the user's browser on login. For more experienced web
developers, this might've been obvious but it was not to me.

To get this working, I added a sidebarCollapsed to the User cookie created on log in / register and added to the user's
browser:

```tsx
export type User = {
  id: string;
  email: string;
  verified: boolean;
  sidebarCollapsed: boolean; // -- HERE --
};

// I use remix-auth for authentication and remix-auth-form for the FormStrategy
export const auth = new Authenticator<User>(sessionStorage);
auth.use(
  new FormStrategy<User>(async ({ form, context }) => {
    // if we have a user in context (since registration creates a user from registration), return it
    if (context?.user) {
      const newUser = context.user as User;
      newUser.sidebarCollapsed = false;
      return newUser;
    }

    const email = form.get("email") as string;
    const password = form.get("password") as string;

    // for login flow
    const user = await prisma.user.findUnique({
      where: { email },
      select: {
        id: true,
        email: true,
        verified: true,
        passwordHash: true,
      },
    });

    if (!user) throw new AuthorizationError("Invalid credentials");

    const isValidPassword = await bcrypt.compare(password, user.passwordHash);
    if (!isValidPassword) throw new AuthorizationError("Invalid credentials");

    return {
      id: user.id,
      email: user.email,
      verified: user.verified,
      sidebarCollapsed: false, // -- HERE --
    };
  })
);
```

That's simple enough.

Since the user has logged in at this point, the `requireUser` function that verifies they're logged in
provides the state we need to power the component that renders our Ui layout, `AppLayout`. This state comes
from the session cookie in the browser:

```tsx
export async function requireUser(request: Request) {
  const user = await auth.isAuthenticated(request);
  if (!user) {
    throw redirect("/auth/login");
  }
  return user;
}
```

To tie it all together, we are missing:
1. A hook to provide the sidebar's current state, and a way to update it.
2. A route to update the cookie in the user's browser.
3. A look at how the hook is used in `AppLayout`

The hook encapsulates code that would have otherwise been in the `AppLayout` component. This code works to initialize,
modify current state and update session cookie information:

```tsx
import { useFetcher } from "@remix-run/react";
import { useCallback, useState } from "react";

export function useSidebarState(initialCollapsed: boolean) {
  const [collapsed, setCollapsed] = useState(initialCollapsed);
  const fetcher = useFetcher();

  const updateCollapsed = useCallback(
    (value: boolean) => {
      setCollapsed(value);

      fetcher.submit(
        { collapsed: value },
        {
          action: "/api/preferences/session",
          method: "POST",
          encType: "application/json",
        }
      );
    },
    [fetcher]
  );

  return [collapsed, updateCollapsed] as const;
}
```

The hook uses a fetcher to perform AJAX-like calls to a provided route. In this case, the route provides a response
to the browser that lets it know to update the session cookie it currently holds:

```tsx
import { ActionFunction, json } from "@remix-run/node";
import { sessionStorage } from "~/services/auth.server";

export const action: ActionFunction = async ({ request }) => {
  const session = await sessionStorage.getSession(
    request.headers.get("Cookie")
  );
  const { collapsed } = await request.json();

  const user = session.get("user");
  if (!user) {
    return json({ error: "No user in session" }, { status: 401 });
  }

  // Update user in session
  user.sidebarCollapsed = collapsed;
  session.set("user", user);
  return json(
    { success: true },
    {
      headers: {
        "Set-Cookie": await sessionStorage.commitSession(session),
      },
    }
  );
};
```

Finally, we can use the hook in our `AppLayout` component like so:

```tsx
// Imports and Types

export function AppLayout({
  children,
  user,
}: {
  children: React.ReactNode;
  user: { email: string; sidebarCollapsed: boolean };
}) {
  const [collapsed, setCollapsed] = useSidebarState(user.sidebarCollapsed);

  return (
    // Rest of component
  );
}
```

So the next time I hit refresh or navigate to a link within my site, I have up-to-date information that is also
accessible to the server for rendering the new page.

*jp*
