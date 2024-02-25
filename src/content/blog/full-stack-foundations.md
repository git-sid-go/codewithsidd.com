---
author: Siddharth Srinivasan
pubDatetime: 2024-02-25T10:34:18Z
title: Full stack foundations
slug: full-stack-foundations
featured: true
tags:
  - summary
description: Full stack foundations with Remix
---

## Table of contents

## links

`links` is a special export in Remix (checkout styling guide in the Remix docs)

This `links` export allow us to define an array of objects which would then be constructed into link tags on the page.

These links can be configured to be present based on whether a route is active or not.

## Responsive favicon

Solution #1: Without Remix

Add the following `link` tag in the `head` section.

```html
<link rel="icon" type="image/svg+xml" href="/favicon.svg" />
```

Solution #2: With Remix

Export the special links object in the component.

```tsx
export const links: LinksFunction = () => {
	return [
		{ rel: 'icon', type: 'image/svg+xml', href: '/favicon.svg' },
	]
}
```

This approach lets us add `links` from any component within the nested routes.

### Asset import caching issue

Let's say we want to update the color of the favicon and we update the svg file. We will notice that we still get the old icon and that is because of the default cache time of 1 hour. 

To fix this, let's add a query param to the icon path whenever we update it. It can be anything and in this example we add `?v=2` (version 2?) to the path.

```tsx
export const links: LinksFunction = () => {
	return [
		{ rel: 'icon', type: 'image/svg+xml', href: '/favicon.svg?v=2' },
	]
}
```

The problem with this approach is that we need to manually update the query param whenever we modify the icon file. We can avoid this with the help of Remix. Let's import the file and add it to the config rather than hardcoding tha path.

```tsx
import faviconAssetUrl from './assets/favicon.svg'

export const links: LinksFunction = () => {
	return [
		{ rel: 'icon', type: 'image/svg+xml', href: faviconAssetUrl },
	]
}
```

With this approach, a fingerprint gets added to the file name based on the contents of the file.

Example:
```ts
favicon-32HPPLKM.svg

favicon-${fingerprint}.svg
```

## Adding global styles to a Remix app

We follow the same approach that we took for the favicon, except that the value of `rel` would be `stylesheet`

```tsx
import faviconAssetUrl from './assets/favicon.svg'
import fontStylesheetUrl from './styles/font.css'

export const links: LinksFunction = () => {
	return [
		{ rel: 'icon', type: 'image/svg+xml', href: faviconAssetUrl },
        { rel: 'stylesheet', href: fontStylesheetUrl },
	]
}
```

## Using Postcss and Tailwind CSS in Remix

### Step 1:

Add PostCSS config. Create a file `postcss.config.js` in the root of remix app with the following contents.

```js
export default {
	plugins: {
		'tailwindcss/nesting': {},
		tailwindcss: {},
		autoprefixer: {},
	},
}
```

### Step 2:

Add Tailwind config. Create a file `tailwind.config.ts` in the root of remix app with the following contents.

```ts
import { type Config } from 'tailwindcss'
import defaultTheme from 'tailwindcss/defaultTheme.js'

export default {
	content: ['./app/**/*.{ts,tsx,jsx,js}'],
	theme: {
		extend: {
			fontFamily: {
				sans: [
					'Nunito Sans',
					'Nunito Sans Fallback',
					...defaultTheme.fontFamily.sans,
				],
			},
		},
	},
} satisfies Config
```

### Step 3:

Create a `tailwind.css` file at `app/styles/tailwind.css` with the following contents.

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### Step 4:

Add the tailwind css file to the `links` export.

```tsx
import faviconAssetUrl from './assets/favicon.svg'
import fontStylesheetUrl from './styles/font.css'
import tailwindStylesheetUrl from './styles/tailwind.css'

export const links: LinksFunction = () => {
	return [
		{ rel: 'icon', type: 'image/svg+xml', href: faviconAssetUrl },
        { rel: 'stylesheet', href: fontStylesheetUrl },
        { rel: 'stylesheet', href: tailwindStylesheetUrl },
	]
}
```

That's all! We should be able to start using Tailwind in our remix app.

## Bundling CSS in Remix

What if we need to add some styles that are provided by some third party libraries we are using. What if there is a way to add a `global.css` file to our app and that will contain all of our common and library specific css?

Remix allows you to import CSS files (side-effect imports) that are bundled automatically.

```ts
import stylesheetUrl from './styles1.css' // <-- you use the URL in the links export
import './global.css' // <-- this will be bundled
```

So, if you just import the CSS file without an "import clause" (the `stylesheetUrl` variable in the example above), it will be bundled for you.

However, we are still responsible for everything on the page between the <html> and the </html> tags, so if we want the bundled CSS file on the page then we need to make sure we add it to the links. Remix gives us access to that URL through a special package called @remix-run/css-bundle:

```ts
import stylesheetUrl from './styles1.css' // <-- you use the URL in the links export
import './global.css' // <-- this will be bundled
import { cssBundleHref } from '@remix-run/css-bundle'

export const links: LinksFunction = () => {
	return [
		...(cssBundleHref ? [{ rel: 'stylesheet', href: cssBundleHref }] : []),
		{ rel: 'icon', type: 'image/svg+xml', href: faviconAssetUrl },
		{ rel: 'stylesheet', href: fontStylesheetUrl },
		{ rel: 'stylesheet', href: tailwindStylesheetUrl },
	]
}
```

### Why can't we just import all of the styles as side-effect imports? Even the `tailwind.css` and `font.css` files?

Reason 1: Any change in any of the files would update the fingerprint of the single `css-bundle...` file which would mean that we do not make use of the cache.

Reason 2:
With the bundling approach, we get control over the order of the CSS files which in turn affects the priority of the styles.


## Further reading

[`link` tag documentation](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/link)

[Using Tailwind with preprocessors](https://tailwindcss.com/docs/using-with-preprocessors)

[CSS bundling in Remix](https://remix.run/docs/en/main/styling/bundling#css-side-effect-imports)

Checkout `modulepreload`