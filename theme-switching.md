# Sveltkit theme switching

###### By Jazza

### What will be covered

In this simple guide, you will learn how to create a theme switcher for your website, render it server-side to avoid a flash when loading the page initially, store the theme in cookies, and use the built-in SvelteKit use:enhance. Note: This guide is made by an inexperienced programmer, and does not feature a complete or polished website.

We will be storing the current theme in the html tag using `document.documentElement.dataset`. In this case, we will really be setting the `data-theme` ourselves, in the `src/app.html` file.

We will begin by adding the data-theme property to the html. Navigate to `src/app.html` and edit the html tag to include `data-theme=""`

Example `src/app.html`:

```html
<!DOCTYPE html>
<html lang="en" data-theme="">
  <head>
    <meta charset="utf-8" />
    <link rel="icon" href="%sveltekit.assets%/favicon.png" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    %sveltekit.head%
  </head>
  <body data-sveltekit-preload-data="hover">
    <div style="display: contents">%sveltekit.body%</div>
  </body>
</html>
```

This gives us a scaffolding to replace with real data later on.

Next, we will be creating the form element which will hold our theme switching. I recommend this to be in the top-level `+layout.svelte` file so that it persists across each page in your website.

Create a `form` element, with no action, as we will be using `formaction` in the child elements. The method will be `POST`, and we will also use sveltekit's `use:enhance` Inside of this form element, for this example we will include two `button` elements. One will switch to the dark theme, and one will switch to the light theme.

```html
<form method="POST" use:enhance="{submitUpdateTheme}">
  <button formaction="/?/setTheme&theme=dark&redirectTo={$page.url.pathname}">
    Dark
  </button>
  <button formaction="/?/setTheme&theme=light&redirectTo={$page.url.pathname}">
    Light
  </button>
</form>
```

`method="POST"` is used to send data to a server, in this case our internal server.
<br>Let's break down the `formaction`:
<br>`/?/`: Allows this to work from any page in your website.
<br>`setTheme`: This is the action that will be receieved by our server.
<br>`&theme=light` and `&redirectTo={see below}`: These are simply url search paramters, which can be accessed by our server.
<br>`{$page.url.pathname}`: This is the pathname of the page we are currently on, and want to redirect back to. Page needs to be defined in a script above, which we will do next.

In order for a few pieces of our form to work, we need to use some JavaScript (or TypeScript). We need to define `enhance` for `use:enhance`, define the function `submitUpdateTheme`, and define `page`.

`enhance` is imported from sveltekit, from `$app/forms`
<br>`page` is imported from sveltekit, from `$app/stores`

`submitUpdateTheme` is a SubmitFunction, so in TS we need to import it's type:
<br>`import type { SubmitFunction } from "@sveltejs/kit";`

Now we define the function:

```ts
const submitUpdateTheme: SubmitFunction = ({ action }) => {
  const theme = action.searchParams.get("theme");

  if (theme) {
    document.documentElement.setAttribute("data-theme", theme);
  }
};
```

Using enhance allows the same behavior of submitting an internal server request without causing the page to reload. See [the docs](https://kit.svelte.dev/docs/form-actions#progressive-enhancement) for more information.

The custom function we used for `user:enhance` allows us to set the `data-theme` of the html element to the desired theme, by using `setAttribute` on the document element. We get the desired theme from the search parameters from the `formaction` url.

While we are still in this file, we will add the CSS to actually switch themes. We will be switching the values of CSS variables, which you will use in every css declaration you want to switch per theme.

At the bottom of the `+layout.svelte` file, add a style tag. In my case I am using SCSS. Then, import a SCSS file relatively (aka using "./" or "../" not "/"). I am importing a SCSS file from the root src of the project, and as we are one level deep, I use "../" to go up one level.

```html
<style lang="scss">
  @import "../global.scss";
</style>
```

Note: This file does not exist yet.

Example `src/routes/+layout.svelte`:

```html
<script lang="ts">
  import { enhance } from "$app/forms";
  import { page } from "$app/stores";

  import type { SubmitFunction } from "@sveltejs/kit";

  const submitUpdateTheme: SubmitFunction = ({ action }) => {
    const theme = action.searchParams.get("theme");

    if (theme) {
      document.documentElement.setAttribute("data-theme", theme);
    }
  };
</script>

<form method="POST" use:enhance="{submitUpdateTheme}">
  <button formaction="/?/setTheme&theme=dark&redirectTo={$page.url.pathname}">
    Dark
  </button>
  <button formaction="/?/setTheme&theme=light&redirectTo={$page.url.pathname}">
    Light
  </button>
</form>

<slot></slot>

<style lang="scss">
  @import "../global.scss";
</style>
```

We will now create the global.scss file. In essence, as we are controlling the theme based off `data-theme` on the html element, we need to target that element in our css selector. We can do this with `html[data-theme]`.

Example `src/global.scss`

```scss
html {
    // This is the default, and hence light, theme.
    --bg: white;
    --text: black;
    --ect-ect;
}

html[data-theme="dark"]{
    --bg: #222;
    --text: white;
    --ect-ect
}
```

When the data-theme is set to "dark" the css variables with be overriden with their new values. As this is imported into the top level `+layout.svelte` these variables will be avaliable all throughout your website.

Next we will create the `hooks.server.ts` file, which intercepts the page load request, and allows us to modify the page the user receives. We will use this to modify the css variables, and send that to the user directly, instead of modifying the page after it has been loaded, for example by using JavaScript.

Example `src/hooks.server.ts`:

```ts
import type { Handle } from "@sveltejs/kit";

export const handle: Handle = async ({ event, resolve }) => {
  let theme: string | null = null;

  const newtheme = event.url.searchParams.get("theme");
  const cookieTheme = event.cookies.get("colortheme");

  if (newtheme) {
    theme = newtheme;
  } else if (cookieTheme) {
    theme = cookieTheme;
  }

  if (theme) {
    return await resolve(event, {
      transformPageChunk: ({ html }) =>
        html.replace(`data-theme=""`, `data-theme="${theme}"`),
    });
  }

  return await resolve(event);
};
```

What we are doing here is receiving the page load event, and getting the desired theme from both the url search paramters (these will only exist if we are switching theme), and from the cookies (the user's previously chosen theme.)
<br>If there is a theme that we need to send back to the user, we resolve the event, but modify the page being sent back. Specifically, we modify the html elemennt, and replace the `data-theme` value from a blank value (taken from `src/app.html`) to the user's desried theme. Critically, this happens server-side, which means the user is served this html, and as such the CSS styles will be put into effect immediately. This is not the case if we ran this code on the client side, as the aJvaScript would need to load first, causing a visible flash.

Finally, we need a `+page.server.ts` file. This file is ran after the server hooks file, and runs server side code before finally sending the page to the user.

Example: `src/routes/+page.server.ts`:

```ts
import { redirect, type Actions } from "@sveltejs/kit";

export const actions: Actions = {
  setTheme: async ({ url, cookies }) => {
    const theme = url.searchParams.get("theme");
    const redirectTo = url.searchParams.get("redirectTo");

    if (theme) {
      cookies.set("colortheme", theme, {
        path: "/",
        maxAge: 60 * 60 * 24 * 365,
      });
    }

    throw redirect(303, redirectTo ?? "/");
  },
};
```

This is the file which contains the server side code. At this point, we run the action `setTheme` as defined in `src/routes/+layout.svelte`. This function gets the theme we are switching to, and sets the user's cookies to the theme. This allows us to use it persistently every time a page loads. It also gets the url we are going to redirect to. Currently, the url would contain `/setTheme&theme=light` and the `redirectTo` parameter, which is messy, so we instead redirect to a clean link.

Hopefully this is all accurate. Thanks for reading!
