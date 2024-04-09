---
title: "WebShot Refactoring"
date: 2024-04-09T19:57:00+01:00
---


## A Journey to Modernization
Recently, while updating my resume, I took a critical look at my [WebShot](https://web-shot.ir) project. The codebase, written in TypeScript with Express for the backend and EJS/SCSS for the frontend, felt outdated compared to modern technologies. While updating was an option, I decided a complete refactor was the best path forward.


## NestJS: A Promising Start
{{<image src="/images/web-shot-refactoring/nestjs.png" alt="NestJS">}}

[NestJS](https://nestjs.com/) immediately caught my eye. Its elegance, rich features, and native TypeScript support made it a strong contender. I started by creating a new project using the NestJS CLI and wrote the controllers with enthusiasm.

However, I encountered a hurdle when it came to serving the WebShot frontend. While NestJS supports [template rendering](https://docs.nestjs.com/techniques/mvc#dynamic-template-rendering), it wouldn't handle my specific needs.  My frontend relied on SCSS and TypeScript, requiring a bundler like Webpack or Vite for effective asset management.  

Having wrestled with bundlers in the past, the prospect of significant time and effort investment became a deal-breaker.  Regretfully, I decided to move on from NestJS and explore other options.

## NuxtJS

As a Vue.js fan, Nuxt.js, a full-stack framework with built-in server-side rendering (SSR), became my top choice. I utilized the Nuxt CLI to establish a new project and began development.
Nuxt's server-side rendering hinges on a component called [Nitro](https://nitro.unjs.io/), previously part of Nuxt itself.

Nitro acts as a standalone webserver for JavaScript runtimes like Node.js.

I effortlessly integrated [Prisma](https://prisma.io/), [Sharp](https://sharp.pixelplumbing.com), and [Puppeteer](https://pptr.dev/) to construct the application.

Nitro's documentation had some gaps, but I was able to find the info I needed with a little extra digging.

### Concurrency
Initially, capturing screenshots might seem like a simple, multi-step process:

1. Launch a browser using Puppeteer.
2. Open a tab and navigate to the user-provided URL.
3. Capture a screenshot and return it.
4. Close the tab and browser.


However, this approach crumbles in a multi-user, high-traffic scenario. Multiple users might request screenshots of different or even identical URLs.
Launching and closing the browser for each capture is inefficient. To optimize resource usage, the browser instance needs to be shared.

Here's where concurrency challenges arise:

{{<mermaid>}}
sequenceDiagram
	autonumber
    actor Bob
    actor Alice
    Alice->>+Browser: Open a tab, go to to google.com
    Browser-->>+Alice: Tab = 1
    Bob->>+Browser: Open a tab, go to to wikipedia.org
    Browser-->>+Bob: Tab = 2
    Alice->>+Browser: Bring forward tab 1
    Browser-->>+Alice: active-tab = 1
    Bob->>+Browser: Bring forward tab 2
    Browser-->>+Bob: active-tab = 2
    Alice->>+Browser: Capture the active tab
    Browser-->>+Alice: Done, image=tab-2.jpg
    Bob->>+Browser: Capture the active tab
    Browser-->>+Bob: Done, image=tab-2.jpg
{{</mermaid>}}

In this scenario, Bob receives an unintended image because his tab activation overlaps with Alice's capture.
To address this, we need to make the capture process atomic.
I implemented a locking mechanism. During tab activation and capture, the browser is locked.
Once the promise resolves, the lock is released, preventing other requests from interfering.

{{<mermaid>}}
sequenceDiagram
	autonumber
    actor Bob
    actor Alice
    participant Lock
    participant Browser

    Alice->>+Browser: Open a tab, go to to google.com
    Browser-->>+Alice: Tab = 1

    Bob->>+Browser: Open a tab, go to to wikipedia.org
    Browser-->>+Bob: Tab = 2

    Alice->>+Lock: Give me browser
    Lock->>+Alice: Browser is yours
    Alice->>+Browser: Bring forward tab 1 and capture

    Bob->>+Lock: Give me browser
    Browser-->>+Alice: Done, image=tab-1.jpg
    Alice->>+Lock: Unlock

    Lock->>+Bob: Browser is yours
    Bob->>+Browser: Bring forward tab 2 and capture
    Browser-->>+Bob: Done, image=tab-2.jpg
    Bob->>+Lock: Unlock
{{</mermaid>}}



### Error Handling

Another backend hurdle involved [Satori](https://github.com/vercel/satori), a library used for generating error messages.
Since WebShot serves screenshots directly within HTML tags or Javascript programs, traditional JSON responses wouldn't suffice.
The errors needed to be embedded within the image itself.

{{<image src="/images/web-shot-refactoring/chrome-like-error.png" alt="Chrome Like Error" >}}

Satori, with its magic (the specifics of which remain a mystery to me!), creates error messages mimicking Google Chrome's style.
As I was using a Vue-based framework, I opted for [v-satori](https://github.com/wobsoriano/v-satori), a Vue adapter for Satori.
v-satori leverages Vue's built-in SSR to transpile components into HTML, which [satori-html](https://github.com/natemoo-re/satori-html) then processes.

{{<mermaid>}}
flowchart LR
    A[Vue Component]-->|v-satori|B[HTML]-->|satori-html|C[React-elements objects]-->|satori|D[SVG]
{{</mermaid>}}


A challenge arose when incorporating local images. To display a local image, its content needs base64 encoding and insertion into the <img> tag's src attribute. 
Otherwise satori going to fetch the image from the internet.
The problem was importing an image in nuxt is not easily possible.
Nuxt relays on [`<NuxtImage />`](https://image.nuxt.com/usage/nuxt-img) component for handling images and this was a edge sitution which was not support.

Through investigation, I discovered Nuxt utilizes Vite for the frontend and [Rollup](https://rollupjs.org/) for Nitro (the backend engine).
Both configurations can be customized within the `nuxt.config.ts` file.
Fortunately, Rollup offers a plugin named @rollup/plugin-image that allows image importing as base64 strings.


{{< tabgroup >}}
{{< tab name="nuxt.config.ts" >}}
```ts
import image from '@rollup/plugin-image';

export default defineNuxtConfig({
	nitro: {
		rollupConfig: {
			plugins: [
				image()
			]
		}
	}
}
```

You can access to complete source code on Github: [nuxt.config.ts](https://github.com/dnj/web-shot/blob/Improve-frontend/nuxt.config.ts)
{{< /tab >}}

{{< tab name="ErrorImage.vue" >}}
```vue
<template>
    <img :src="img" :width="74" :height="74" />
</template>
<script>
import img from "@/assets/images/error/dinasor.png";
export default defineComponent({
    setup() {
        return { img };
    },
});
</script>
```
You can access to complete source code on Github: [ErrorImage.vue](https://github.com/dnj/web-shot/blob/Improve-frontend/components/ErrorImage.vue)
{{< /tab >}}
{{< /tabgroup >}}


This blog post is the story of WebShot's makeover: choosing Nuxt.js, conquering concurrency challenges, and mastering error handling with Satori.

It wasn't always smooth sailing, but the lessons learned were epic!