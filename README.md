# DiscordStrategy

The Discord strategy is used to authenticate users against a Discord account. It extends the OAuth2Strategy.

## Usage

### Create an OAuth application

First go to [the Discord Developer Portal](https://discord.com/developers/applications) to create a new application and get a client ID and secret. The client ID and secret are located in the OAuth2 Tab of your Application. Once you are there you can already add your first redirect url, f.e. `http://localhost:3000/auth/discord/callback`.

You can find the detailed Discord OAuth Documentation [here](https://discord.com/developers/docs/topics/oauth2#oauth2).

### Create your session storage

```ts
// app/session.server.ts
import { createCookieSessionStorage } from "@remix-run/node";

export const sessionStorage = createCookieSessionStorage({
  cookie: {
    name: "_session",
    sameSite: "lax",
    path: "/",
    httpOnly: true,
    secrets: ["s3cr3t"],
    secure: process.env.NODE_ENV === "production",
  },
});

export const { getSession, commitSession, destroySession } = sessionStorage;
```

### Create the strategy instance

```ts
// app/auth.server.ts
import { Authenticator } from "remix-auth";
import type { DiscordProfile, PartialDiscordGuild } from "remix-auth-discord";
import { DiscordStrategy } from "remix-auth-discord";
import { sessionStorage } from "~/session.server";

export interface DiscordUser {
  id: DiscordProfile["id"];
  displayName: DiscordProfile["displayName"];
  avatar: DiscordProfile["__json"]["avatar"];
  discriminator: DiscordProfile["__json"]["discriminator"];
  email: DiscordProfile["__json"]["email"];
  guilds?: Array<PartialDiscordGuild>;
  accessToken: string;
  refreshToken: string;
}

export const auth = new Authenticator<DiscordUser>(sessionStorage);

const discordStrategy = new DiscordStrategy(
  {
    clientID: "YOUR_CLIENT_ID",
    clientSecret: "YOUR_CLIENT_SECRET",
    callbackURL: "https://example.com/auth/discord/callback",
    // Provide all the scopes you want as an array
    scope: ["identify", "email", "guilds"],
  },
  async ({
    accessToken,
    refreshToken,
    extraParams,
    profile,
  }): Promise<DiscordUser> => {
    /**
     * Get the user data from your DB or API using the tokens and profile
     * For example query all the user guilds
     * IMPORTANT: This can quickly fill the session storage to be too big.
     * So make sure you only return the values from the guilds (and the guilds) you actually need
     * (eg. omit the features)
     * and if that's still to big, you need to store the guilds some other way. (Your own DB)
     *
     * Either way, this is how you could retrieve the user guilds.
     */
    const userGuilds: Array<PartialDiscordGuild> = await (
      await fetch("https://discord.com/api/v10/users/@me/guilds", {
        headers: {
          Authorization: `Bearer ${accessToken}`,
        },
      })
    )?.json();
    /**
     * In this example we're only interested in guilds where the user is either the owner or has the `MANAGE_GUILD` permission (This check includes the `ADMINISTRATOR` permission)
     */
    const guilds: Array<PartialDiscordGuild> = userGuilds.filter(
      (g) => g.owner || (BigInt(g.permissions) & BigInt(0x20)) == BigInt(0x20)
    );

    /**
     * Construct the user profile to your liking by adding data you fetched etc.
     * and only returning the data that you actually need for your application.
     */
    return {
      id: profile.id,
      displayName: profile.__json.username,
      avatar: profile.__json.avatar,
      discriminator: profile.__json.discriminator,
      email: profile.__json.email,
      accessToken,
      guilds,
      refreshToken,
    };
  }
);

auth.use(discordStrategy);
```

### Setup your routes

```tsx
// app/routes/login.tsx
import { Form } from "@remix-run/react";

export default function Login() {
  return (
    <Form action="/auth/discord" method="post">
      <button>Login with Discord</button>
    </Form>
  );
}
```

```tsx
// app/routes/auth/discord.tsx
import type { ActionFunction, LoaderFunction } from "@remix-run/node";
import { redirect } from "@remix-run/node";
import { auth } from "~/auth.server";

export let loader: LoaderFunction = () => redirect("/login");

export let action: ActionFunction = ({ request }) => {
  return auth.authenticate("discord", request);
};
```

```tsx
// app/routes/auth/discord.callback.tsx
import type { LoaderFunction } from "@remix-run/node";
import { auth } from "~/auth.server";

export let loader: LoaderFunction = ({ request }) => {
  return auth.authenticate("discord", request, {
    successRedirect: "/dashboard",
    failureRedirect: "/login",
  });
};
```

```tsx
// app/routes/dashboard.tsx
import type { LoaderFunction } from "@remix-run/node";
import { useLoaderData } from "@remix-run/react";
import { auth } from "~/auth.server";
import type { DiscordUser } from "~/auth.server";

export let loader: LoaderFunction = async ({ request }) => {
  return await auth.isAuthenticated(request, {
    failureRedirect: "/login",
  });
};

export default function DashboardPage() {
  const user = useLoaderData<DiscordUser>();
  return (
    <div>
      <h1>Dashboard</h1>
      <h2>
        Welcome {user.displayName}#{user.discriminator}
      </h2>
    </div>
  );
}
```

That's it, try going to `/login` and press the Login button to start the authentication flow. Make sure to store all your Secrets properly and setup the correct redirect_url once you go to production.
