+++
title = "How to use Google Sign In with Vue"
description = """
  A quick overview of how I managed to use Google Sign In with Vue (Nuxt)
"""
type = ["posts","post"]
tags = [
    "vue",
    "nuxt",
    "google",
    "oauth",
    "ui"
]
date = "2024-03-25T00:00:00+01:00"
categories = [
    "Web Development",
]
+++
## TL;DR

```jsx
<template>
  <component is="script" src="https://accounts.google.com/gsi/client" @load="initSignIn" async />
  <div id="gSignInButton" />
</template>

<script setup lang="ts">
const loginCallback = async (response) => {
  // send response to your own server endpoint
  // and redirect user
};

const initSignIn = () => {
  google.accounts.id.initialize({
    client_id: "YOUR_CLIENT_ID",
    callback: loginCallback
  });

  google.accounts.id.renderButton(
    document.getElementById("gSignInButton"),
    { type: "standard", text: "sign_in_with", theme: "outline",
      size: "large", width: "400" }
  );
  google.accounts.id.prompt();
};

onMounted(() => {
  if (typeof google !== 'undefined') {
    initSignIn();
  }
});
</script>
```

## Introduction

A common situation when building a website is letting users sign in with their preexisting
Google accounts. Google has made this process rather simple, compared to the traditional
OAuth provider set up, with documentation available [here][google-button].

However, the documentation only covers vanilla JS and HTML (why?).
Making the sign in prompt and button work in Vue took a little bit more work.

> This guide will not go through the process of obtaining a Google API Client ID,
> for that you can read Google's documentation.

## First Steps

Right off the bat Google gives us two alternatives: use HTML or JavaScript to display the
login button. The HTML approach actually still uses JavaScript, only that the process is
automated by using a predetermined button `id`.

Regardless of which approach you use, you must first include Google's Sign In script,
available at `https://accounts.google.com/gsi/client`.
The naive thing is to try to use the `script` tag, as it appears in Google's documentation.

```html
<script src="https://accounts.google.com/gsi/client" async />
```
However as of Vue 3, using raw script tags within a component is not allowed.
Instead, you can use the following approach:

```html
<component is="script" src="https://accounts.google.com/gsi/client" async />
```

With this sorted out, it's time review our approaches.


## HTML Approach

With the `script` tag adapted to work in Vue 3, we can already create a basic Vue
component using the HTML approach.

```html
<template>
<div id="login-provider-google">
  <component is="script" src="https://accounts.google.com/gsi/client" async />
  <div id="g_id_onload"
    data-client_id="..."
    data-context="signin"
    data-ux_mode="popup"
    data-login_uri="..."
    data-itp_support="true">
  </div>

  <div class="g_id_signin"
    data-type="standard"
    data-shape="rectangular"
    data-theme="outline"
    data-text="signin_with"
    data-size="large"
    data-logo_alignment="left">
  </div>
</div>
</template>
```

This component uses a redirect, rather than a callback function.
If you are happy with using a redirect, this is the approach for you.

So why not simply use the `data-callback` attribute (as per documentation) and pass in
a callback too? I tried to, but I found it hard to directly pass a handler defined within
the component. I'm sure it is possible, but I had more important things to do than to find
workarounds for this.

Instead I resorted to the other approach.

## JS Approach

In this approach, you are tasked with defining your own google sign in button element
and later initialise it within JavaScript.
We can easily replace the window.onload listener in the documentation with an
`onMounted` Vue hook.

```jsx
<template>
  <component is="script" src="https://accounts.google.com/gsi/client" async />
  <div id="gSignInButton" />
</template>

<script setup lang="ts">
const initSignIn = () => {
  google.accounts.id.initialize({
    client_id: "YOUR_CLIENT_ID",
    callback: (response) => { // do something }
  });

  google.accounts.id.renderButton(
    document.getElementById("gSignInButton"),
    { type: "standard", text: "sign_in_with", theme: "outline",
      size: "large", width: "400" }
  );
  google.accounts.id.prompt();
};

onMounted(() => {
  initSignIn();
});
</script>
```

This works for the most part. But there will be times when the script has not loaded
by the time the onMounted hook is executed. Even removing the async attribute won't fix
it. This leads to the very confusing effect of having a login page with no button.
This may be somewhat hard to catch too. I was only able to trigger this by logging in,
then visiting a few pages, reloading, and then signing out. When I was redirected
to the login page, as clockwork, there was no Google Sign In button.

To account for this, we can take advantage of the onLoad event, which triggers once a script
is finally loaded.

```jsx
<component is="script" src="https://accounts.google.com/gsi/client" @load="initSignIn" async />
```

Note that we have to leave the `onMounted` hook in place, for if the script has already loaded
the event will not trigger. To avoid any errors, we can first check if the library has already
loaded.

```js
onMounted(() => {
  if (typeof google !== 'undefined') {
    initSignIn();
  }
});
```

That should cover everything needed to use the Google Sign In button in Vue.
You could add your own logic to easily switch between the log-in and sign-up layouts, amongst
other things.

## FAQ

- Why not execute the script on the fly, for instance by adding the script element
  after the fact?

It just seemed a bit of a hack, and there was ultimately no need to.

- Why not use 'vue-plugin-load-script' instead?

Same as last question. Besides, it hasn't seen a commit in several years.
The last thing you want is to have unmaintained dependencies in an active project.

<!-- LINKS -->
[google-button]: https://developers.google.com/identity/gsi/web/guides/display-button
