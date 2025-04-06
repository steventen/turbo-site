---
permalink: /handbook/introduction.html
redirect_from: /handbook/
description: "Turbo bundles several techniques for creating fast, modern web applications without having to reach for a client-side JavaScript framework."
---

# Introduction

Turbo bundles several techniques for creating fast, modern, progressively enhanced web applications without using much JavaScript. It offers a simpler alternative to the prevailing client-side frameworks which put all the logic in the front-end and confine the server side of your app to being little more than a JSON API.

With Turbo, you let the server deliver HTML directly, which means all the logic for checking permissions, interacting directly with your domain model, and everything else that goes into programming an application can happen more or less exclusively within your favorite programming language. You're no longer mirroring logic on both sides of a JSON divide. All the logic lives on the server, and the browser deals just with the final HTML.

You can read more about the benefits of this HTML-over-the-wire approach on the <a href="https://hotwired.dev/">Hotwire site</a>. What follows are the techniques that Turbo brings to make this possible.

## Turbo Drive: Navigate within a persistent process

A key attraction with traditional single-page applications, when compared with the old-school, separate-pages approach, is the speed of navigation. SPAs get a lot of that speed from not constantly tearing down the application process, only to reinitialize it on the very next page.

Turbo Drive gives you that same speed by using the same persistent-process model, but without requiring you to craft your entire application around the paradigm. There's no client-side router to maintain, there's no state to carefully manage. The persistent process is managed by Turbo, and you write your server-side code as though you were living back in the early aughts – blissfully isolated from the complexities of today's SPA monstrosities!

This happens by intercepting all clicks on `<a href>` links to the same domain. When you click an eligible link, Turbo Drive prevents the browser from following it, changes the browser’s URL using the <a href="https://developer.mozilla.org/en-US/docs/Web/API/History">History API</a>, requests the new page using <a href="https://developer.mozilla.org/en-US/docs/Web/API/fetch">`fetch`</a>, and then renders the HTML response.

Same deal with forms. Their submissions are turned into `fetch` requests from which Turbo Drive will follow the redirect and render the HTML response.

During rendering, Turbo Drive replaces the contents of the `<body>` element and merges the contents of the `<head>` element. The JavaScript window and document objects, and the `<html>` element, persist from one rendering to the next.

While it's possible to interact directly with Turbo Drive to control how visits happen or hook into the lifecycle of the request, the majority of the time this is a drop-in replacement where the speed is free just by adopting a few conventions.


## Turbo Frames: Decompose complex pages

Most web applications present pages that contain several independent segments. For a discussion page, you might have a navigation bar on the top, a list of messages in the center, a form at the bottom to add a new message, and a sidebar with related topics. Generating this discussion page normally means generating each segment in a serialized manner, piecing them all together, then delivering the result as a single HTML response to the browser.

With Turbo Frames, you can place those independent segments inside frame elements that can scope their navigation and be lazily loaded. Scoped navigation means all interaction within a frame, like clicking links or submitting forms, happens within that frame, keeping the rest of the page from changing or reloading.

To wrap an independent segment in its own navigation context, enclose it in a `<turbo-frame>` tag. For example:

```html
<turbo-frame id="new_message">
  <form action="/messages" method="post">
    ...
  </form>
</turbo-frame>
```

When you submit the form above, Turbo extracts the matching `<turbo-frame id="new_message">` element from the redirected HTML response and swaps its content into the existing `new_message` frame element. The rest of the page stays just as it was.

Frames can also defer loading their contents in addition to scoping navigation. To defer loading a frame, add a `src` attribute whose value is the URL to be automatically loaded. As with scoped navigation, Turbo finds and extracts the matching frame from the resulting response and swaps its content into place:

```html
<turbo-frame id="messages" src="/messages">
  <p>This message will be replaced by the response from /messages.</p>
</turbo-frame>
```

This may sound a lot like old-school frames, or even `<iframe>`s, but Turbo Frames are part of the same DOM, so there's none of the weirdness or compromises associated with "real" frames. Turbo Frames are styled by the same CSS, part of the same JavaScript context, and are not placed under any additional content security restrictions.

In addition to turning your segments into independent contexts, Turbo Frames affords you:

1. **Efficient caching.** In the discussion page example above, the related topics sidebar needs to expire whenever a new related topic appears, but the list of messages in the center does not. When everything is just one page, the whole cache expires as soon as any of the individual segments do. With frames, each segment is cached independently, so you get longer-lived caches with fewer dependent keys.
1. **Parallelized execution.** Each defer-loaded frame is generated by its own HTTP request/response, which means it can be handled by a separate process. This allows for parallelized execution without having to manually manage the process. A complicated composite page that takes 400ms to complete end-to-end can be broken up with frames where the initial request might only take 50ms, and each of three defer-loaded frames each take 50ms. Now the whole page is done in 100ms because the three frames each taking 50ms run concurrently rather than sequentially.
1. **Ready for mobile.** In mobile apps, you usually can't have big, complicated composite pages. Each segment needs a dedicated screen. With an application built using Turbo Frames, you've already done this work of turning the composite page into segments. These segments can then appear in native sheets and screens without alteration (since they all have independent URLs).


## Turbo Streams: Deliver live page changes

Making partial page changes in response to asynchronous actions is how we make the application feel alive. While Turbo Frames give us such updates in response to direct interactions within a single frame, Turbo Streams let us change any part of the page in response to updates sent over a WebSocket connection, SSE or other transport. (Think an <a href="http://itsnotatypo.com">imbox</a> that automatically updates when a new email arrives.)

Turbo Streams introduces a `<turbo-stream>` element with nine basic actions: `append`, `prepend`, `replace`, `update`, `remove`, `before`, `after`, `morph`, and `refresh`. With these actions, along with the `target` attribute specifying the ID of the element you want to operate on, you can encode all the mutations needed to refresh the page. You can even combine several stream elements in a single stream message. Simply include the HTML you're interested in inserting or replacing in a <a href="https://developer.mozilla.org/en-US/docs/Web/HTML/Element/template">template tag</a> and Turbo does the rest:

```html
<turbo-stream action="append" target="messages">
  <template>
    <div id="message_1">My new message!</div>
  </template>
</turbo-stream>
```

This stream element will take the `div` with the new message and append it to the container with the ID `messages`. It's just as simple to replace an existing element:

```html
<turbo-stream action="replace" target="message_1">
  <template>
    <div id="message_1">This changes the existing message!</div>
  </template>
</turbo-stream>
```

This is a conceptual continuation of what in the Rails world was first called <a href="https://weblog.rubyonrails.org/2006/3/28/rails-1-1-rjs-active-record-respond_to-integration-tests-and-500-other-things/">RJS</a> and then called <a href="https://signalvnoise.com/posts/3697-server-generated-javascript-responses">SJR</a>, but realized without any need for JavaScript. The benefits remain the same:

1. **Reuse the server-side templates**: Live page changes are generated using the same server-side templates that were used to create the first-load page.
1. **HTML over the wire**: Since all we're sending is HTML, you don't need any client-side JavaScript (beyond Turbo, of course) to process the update. Yes, the HTML payload might be a tad larger than a comparable JSON, but with gzip, the difference is usually negligible, and you save all the client-side effort it takes to fetch JSON and turn it into HTML.
1. **Simpler control flow**: It's really clear to follow what happens when messages arrive on the WebSocket, SSE or in response to form submissions. There's no routing, event bubbling, or other indirection required. It's just the HTML to be changed, wrapped in a single tag that tells us how.

Now, unlike RJS and SJR, it's not possible to call custom JavaScript functions as part of a Turbo Streams action. But this is a feature, not a bug. Those techniques can easily end up producing a tangled mess when way too much JavaScript is sent along with the response. Turbo focuses squarely on just updating the DOM, and then assumes you'll connect any additional behavior using <a href="https://stimulus.hotwired.dev">Stimulus</a> actions and lifecycle callbacks.


## Turbo Native: Hybrid apps for iOS & Android

Turbo Native is ideal for building hybrid apps for iOS and Android. You can use your existing server-rendered HTML to get baseline coverage of your app's functionality in a native wrapper. Then you can spend all the time you saved on making the few screens that really benefit from high-fidelity native controls even better.

An application like Basecamp has hundreds of screens. Rewriting every single one of those screens would be an enormous task with very little benefit. Better to reserve the native firepower for high-touch interactions that really demand the highest fidelity. Something like the "New For You" inbox in Basecamp, for example, where we use swipe controls that need to feel just right. But most pages, like the one showing a single message, wouldn't really be any better if they were completely native.

Going hybrid doesn't just speed up your development process, it also gives you more freedom to upgrade your app without going through the slow and onerous app store release processes. Anything that's done in HTML can be changed in your web application, and instantly be available to all users. No waiting for Big Tech to approve your changes, no waiting for users to upgrade.

Turbo Native assumes you're using the recommended development practices available for iOS and Android. This is not a framework that abstracts native APIs away or even tries to let your native code be shareable between platforms. The part that's shareable is the HTML that's rendered server-side. But the native controls are written in the recommended native APIs.

See the <a href="https://github.com/hotwired/turbo-ios">Turbo Native: iOS</a> and <a href="https://github.com/hotwired/turbo-android">Turbo Native: Android</a> repositories for more documentation. See the native apps for HEY on <a href="https://apps.apple.com/us/app/hey-email/id1506603805">iOS</a> and <a href="https://play.google.com/store/apps/details?id=com.basecamp.hey&hl=en_US&gl=US">Android</a> to get a feel for just how good you can make a hybrid app powered by Turbo.


## Integrate with backend frameworks

You don't need any backend framework to use Turbo. All the features are built to be used directly, without further abstractions. But if you have the opportunity to use a backend framework that's integrated with Turbo, you'll find life a lot simpler. [We've created a reference implementation for such an integration for Ruby on Rails](https://github.com/hotwired/turbo-rails).
---
permalink: /handbook/drive.html
description: "Turbo Drive accelerates links and form submissions by negating the need for full page reloads."
---

# Navigate with Turbo Drive

Turbo Drive is the part of Turbo that enhances page-level navigation. It watches for link clicks and form submissions, performs them in the background, and updates the page without doing a full reload. It's the evolution of a library previously known as [Turbolinks](https://github.com/turbolinks/turbolinks).

${toc}

## Page Navigation Basics

Turbo Drive models page navigation as a *visit* to a *location* (URL) with an *action*.

Visits represent the entire navigation lifecycle from click to render. That includes changing browser history, issuing the network request, restoring a copy of the page from cache, rendering the final response, and updating the scroll position.

During rendering, Turbo Drive replaces the contents of the requesting document's `<body>` with the contents of the response document's `<body>`, merges the contents of their `<head>`s too, and updates the `lang` attribute of the `<html>` element as needed. The point of merging instead of replacing the `<head>` elements is that if `<title>` or `<meta>` tags change, say, they will be updated as expected, but if links to assets are the same, they won't be touched and therefore the browser won't process them again.

There are two types of visit: an _application visit_, which has an action of _advance_ or _replace_, and a _restoration visit_, which has an action of _restore_.

## Application Visits

Application visits are initiated by clicking a Turbo Drive-enabled link, or programmatically by calling [`Turbo.visit(location)`](/reference/drive#turbodrivevisit).

An application visit always issues a network request. When the response arrives, Turbo Drive renders its HTML and completes the visit.

If possible, Turbo Drive will render a preview of the page from cache immediately after the visit starts. This improves the perceived speed of frequent navigation between the same pages.

If the visit’s location includes an anchor, Turbo Drive will attempt to scroll to the anchored element. Otherwise, it will scroll to the top of the page.

Application visits result in a change to the browser’s history; the visit’s _action_ determines how.

![Advance visit action](https://s3.amazonaws.com/turbolinks-docs/images/advance.svg)

The default visit action is _advance_. During an advance visit, Turbo Drives pushes a new entry onto the browser’s history stack using [`history.pushState`](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState).

Applications using the Turbo Drive [iOS adapter](https://github.com/hotwired/turbo-ios) typically handle advance visits by pushing a new view controller onto the navigation stack. Similarly, applications using the [Android adapter](https://github.com/hotwired/turbo-android) typically push a new activity onto the back stack.

![Replace visit action](https://s3.amazonaws.com/turbolinks-docs/images/replace.svg)

You may wish to visit a location without pushing a new history entry onto the stack. The _replace_ visit action uses [`history.replaceState`](https://developer.mozilla.org/en-US/docs/Web/API/History/replaceState) to discard the topmost history entry and replace it with the new location.

To specify that following a link should trigger a replace visit, annotate the link with `data-turbo-action="replace"`:

```html
<a href="/edit" data-turbo-action="replace">Edit</a>
```

To programmatically visit a location with the replace action, pass the `action: "replace"` option to `Turbo.visit`:

```js
Turbo.visit("/edit", { action: "replace" })
```

Applications using the Turbo Drive [iOS adapter](https://github.com/hotwired/turbo-ios) typically handle replace visits by dismissing the topmost view controller and pushing a new view controller onto the navigation stack without animation.

## Restoration Visits

Turbo Drive automatically initiates a restoration visit when you navigate with the browser’s Back or Forward buttons. Applications using the [iOS](https://github.com/hotwired/turbo-ios) or [Android](https://github.com/hotwired/turbo-android) adapters initiate a restoration visit when moving backward in the navigation stack.

![Restore visit action](https://s3.amazonaws.com/turbolinks-docs/images/restore.svg)

If possible, Turbo Drive will render a copy of the page from cache without making a request. Otherwise, it will retrieve a fresh copy of the page over the network. See [Understanding Caching](/handbook/building#understanding-caching) for more details.

Turbo Drive saves the scroll position of each page before navigating away and automatically returns to this saved position on restoration visits.

Restoration visits have an action of _restore_ and Turbo Drive reserves them for internal use. You should not attempt to annotate links or invoke `Turbo.visit` with an action of `restore`.

## Canceling Visits Before They Start

Application visits can be canceled before they start, regardless of whether they were initiated by a link click or a call to [`Turbo.visit`](/reference/drive#turbovisit).

Listen for the `turbo:before-visit` event to be notified when a visit is about to start, and use `event.detail.url` (or `$event.originalEvent.detail.url`, when using jQuery) to check the visit’s location. Then cancel the visit by calling `event.preventDefault()`.

Restoration visits cannot be canceled and do not fire `turbo:before-visit`. Turbo Drive issues restoration visits in response to history navigation that has *already taken place*, typically via the browser’s Back or Forward buttons.

## Custom Rendering

Applications can customize the rendering process by adding a document-wide `turbo:before-render` event listener and overriding the `event.detail.render` property.

For example, you could merge the response document's `<body>` element into the requesting document's `<body>` element with [idiomorph](https://github.com/bigskysoftware/idiomorph) or [morphdom](https://github.com/patrick-steele-idem/morphdom):

```javascript
import { Idiomorph } from "idiomorph"

addEventListener("turbo:before-render", (event) => {
  event.detail.render = (currentElement, newElement) => {
    Idiomorph.morph(currentElement, newElement)
  }
})
```

## Pausing Rendering

Applications can pause rendering and make additional preparations before continuing.

Listen for the `turbo:before-render` event to be notified when rendering is about to start, and pause it using `event.preventDefault()`. Once the preparation is done continue rendering by calling `event.detail.resume()`.

An example use case is adding exit animation for visits:
```javascript
document.addEventListener("turbo:before-render", async (event) => {
  event.preventDefault()

  await animateOut()

  event.detail.resume()
})
```

## Pausing Requests

Application can pause request and make additional preparation before it will be executed.

Listen for the `turbo:before-fetch-request` event to be notified when a request is about to start, and pause it using `event.preventDefault()`. Once the preparation is done continue request by calling `event.detail.resume()`.

An example use case is setting `Authorization` header for the request:
```javascript
document.addEventListener("turbo:before-fetch-request", async (event) => {
  event.preventDefault()

  const token = await getSessionToken(window.app)
  event.detail.fetchOptions.headers["Authorization"] = `Bearer ${token}`

  event.detail.resume()
})
```

## Performing Visits With a Different Method

By default, link clicks send a `GET` request to your server. But you can change this with `data-turbo-method`:

```html
<a href="/articles/54" data-turbo-method="delete">Delete the article</a>
```

You should consider that for accessibility reasons, it's better to use actual forms and buttons for anything that's not a GET.

## Requiring Confirmation for a Visit

Decorate links with both `data-turbo-confirm` and `data-turbo-method`, and confirmation will be required for a visit to proceed.

```html
<a href="/articles" data-turbo-method="get" data-turbo-confirm="Do you want to leave this page?">Back to articles</a>
<a href="/articles/54" data-turbo-method="delete" data-turbo-confirm="Are you sure you want to delete the article?">Delete the article</a>
```

Use `Turbo.config.forms.confirm = confirmMethod` to change the method that gets called for confirmation. The default is the browser's built in `confirm`.


## Disabling Turbo Drive on Specific Links or Forms

Turbo Drive can be disabled on a per-element basis by annotating the element or any of its ancestors with `data-turbo="false"`.

```html
<a href="/" data-turbo="false">Disabled</a>

<form action="/messages" method="post" data-turbo="false">
  <!-- … -->
</form>

<div data-turbo="false">
  <a href="/">Disabled</a>
  <form action="/messages" method="post">
    <!-- … -->
  </form>
</div>
```

To reenable when an ancestor has opted out, use `data-turbo="true"`:

```html
<div data-turbo="false">
  <a href="/" data-turbo="true">Enabled</a>
</div>
```

Links or forms with Turbo Drive disabled will be handled normally by the browser.

If you want Drive to be opt-in rather than opt-out, then you can set `Turbo.session.drive = false`; then, `data-turbo="true"` is used to enable Drive on a per-element basis. If you're importing Turbo in a JavaScript pack, you can do this globally:

```js
import { Turbo } from "@hotwired/turbo-rails"
Turbo.session.drive = false
```

## View transitions

In [browsers that support](https://caniuse.com/?search=View%20Transition%20API) the [View Transition API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transitions_API) Turbo can trigger view transitions when navigating between pages.

Turbo triggers a view transition when both the current and the next page have this meta tag:

```
<meta name="view-transition" content="same-origin" />
```

Turbo also adds a `data-turbo-visit-direction` attribute to the `<html>` element to indicate the direction of the transition. The attribute can have one of the following values:

- `forward` in advance visits.
- `back` in restoration visits.
- `none` in replace visits.

You can use this attribute to customize the animations that are performed during a transition:

```css
html[data-turbo-visit-direction="forward"]::view-transition-old(sidebar):only-child {
  animation: slide-to-right 0.5s ease-out;
}
```

## Displaying Progress

During Turbo Drive navigation, the browser will not display its native progress indicator. Turbo Drive installs a CSS-based progress bar to provide feedback while issuing a request.

The progress bar is enabled by default. It appears automatically for any page that takes longer than 500ms to load. (You can change this delay with the [`Turbo.setProgressBarDelay`](/reference/drive#turbodrivesetprogressbardelay) method.)

The progress bar is a `<div>` element with the class name `turbo-progress-bar`. Its default styles appear first in the document and can be overridden by rules that come later.

For example, the following CSS will result in a thick green progress bar:

```css
.turbo-progress-bar {
  height: 5px;
  background-color: green;
}
```

To disable the progress bar entirely, set its `visibility` style to `hidden`:

```css
.turbo-progress-bar {
  visibility: hidden;
}
```

In tandem with the progress bar, Turbo Drive will also toggle the [`[aria-busy]` attribute][aria-busy] on the page's `<html>` element during page navigations started from Visits or Form Submissions. Turbo Drive will set `[aria-busy="true"]` when the navigation begins, and will remove the `[aria-busy]` attribute when the navigation completes.

[aria-busy]: https://www.w3.org/TR/wai-aria/#aria-busy

## Reloading When Assets Change

As we saw above, Turbo Drive merges the contents of the `<head>` elements. However, if CSS or JavaScript change, that merge would evaluate them on top of the existing one. Typically, this would lead to undesirable conflicts. In such cases, it's necessary to fetch a completely new document through a standard, non-Ajax request.

To accomplish this, just annotate those asset elements with `data-turbo-track="reload"` and include a version identifier in your asset URLs. The identifier could be a number, a last-modified timestamp, or better, a digest of the asset’s contents, as in the following example.

```html
<head>
  <!-- … -->
  <link rel="stylesheet" href="/application-258e88d.css" data-turbo-track="reload">
  <script src="/application-cbd3cd4.js" data-turbo-track="reload"></script>
</head>
```

## Removing Assets When They Change

As we saw above, Turbo Drive merges the contents of the `<head>` elements. When a page depends on external assets like CSS stylesheets that other pages do not, it can be useful to remove them when navigating away from the page.

Rendering a `<link>` or `<style>` element with `[data-turbo-track="dynamic"]` instructs Turbo Drive to dynamically remove the element when it is absent from a navigation's response, and can serve a complementary role to the [`[data-turbo-track="reload"]`](#reload-when-assets-change) attribute to avoid triggering a full page reload when deploying changes that only affect styles.

```html
<head>
  <!-- … -->
  <link rel="stylesheet" href="/page-specific-styles-258e88d.css" data-turbo-track="dynamic">
  <style data-turbo-track="dynamic">
    .page-specific-styles { /* … */ }
  </style>
</head>
```

Note that rendering `<script>` elements with `[data-turbo-track="dynamic"]` might have unintended side-effects. When `<script>` disconnected from the document, the JavaScript context doesn't change, nor is the element's already evaluated JavaScript code unloaded or changed in any way.

## Ensuring Specific Pages Trigger a Full Reload

You can ensure visits to a certain page will always trigger a full reload by including a `<meta name="turbo-visit-control">` element in the page’s `<head>`.

```html
<head>
  <!-- … -->
  <meta name="turbo-visit-control" content="reload">
</head>
```

This setting may be useful as a workaround for third-party JavaScript libraries that don’t interact well with Turbo Drive page changes.

## Setting a Root Location

Turbo Drive only loads URLs with the same origin—i.e. the same protocol, domain name, and port—as the current document. A visit to any other URL falls back to a full page load.

In some cases, you may want to further scope Turbo Drive to a path on the same origin. For example, if your Turbo Drive application lives at `/app`, and the non-Turbo Drive help site lives at `/help`, links from the app to the help site shouldn’t use Turbo Drive.

Include a `<meta name="turbo-root">` element in your pages’ `<head>` to scope Turbo Drive to a particular root location. Turbo Drive will only load same-origin URLs that are prefixed with this path.

```html
<head>
  <!-- … -->
  <meta name="turbo-root" content="/app">
</head>
```

## Form Submissions

Turbo Drive handles form submissions in a manner similar to link clicks. The key difference is that form submissions can issue stateful requests using the HTTP POST method, while link clicks only ever issue stateless HTTP GET requests.

Throughout a submission, Turbo Drive will dispatch a series of [events][] that
target the `<form>` element and [bubble up][] through the document:

1. `turbo:submit-start`
2. `turbo:before-fetch-request`
3. `turbo:before-fetch-response`
4. `turbo:submit-end`

During a submission, Turbo Drive will set the "submitter" element's [disabled][] attribute when the submission begins, then remove the attribute after the submission ends. When submitting a `<form>` element, browsers will treat the `<input type="submit">` or `<button>` element that initiated the submission as the [submitter][]. To submit a `<form>` element programmatically, invoke the [HTMLFormElement.requestSubmit()][] method and pass an `<input type="submit">` or `<button>` element as an optional parameter.

If there are other changes you'd like to make during a `<form>` submission (for
example, disabling _all_ [fields within a submitted `<form>`][elements]), you
can declare your own event listeners:

```js
addEventListener("turbo:submit-start", ({ target }) => {
  for (const field of target.elements) {
    field.disabled = true
  }
})
```

[events]: /reference/events
[bubble up]: https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Building_blocks/Events#event_bubbling
[elements]: https://developer.mozilla.org/en-US/docs/Web/API/HTMLFormElement/elements
[disabled]: https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/disabled
[submitter]: https://developer.mozilla.org/en-US/docs/Web/API/SubmitEvent/submitter
[HTMLFormElement.requestSubmit()]: https://developer.mozilla.org/en-US/docs/Web/API/HTMLFormElement/requestSubmit

## Redirecting After a Form Submission

After a stateful request from a form submission, Turbo Drive expects the server to return an [HTTP 303 redirect response](https://en.wikipedia.org/wiki/HTTP_303), which it will then follow and use to navigate and update the page without reloading.

The exception to this rule is when the response is rendered with either a 4xx or 5xx status code. This allows form validation errors to be rendered by having the server respond with `422 Unprocessable Content` and a broken server to display a "Something Went Wrong" screen on a `500 Internal Server Error`.

The reason Turbo doesn't allow regular rendering on 200's from POST requests is that browsers have built-in behavior for dealing with reloads on POST visits where they present a "Are you sure you want to submit this form again?" dialogue that Turbo can't replicate. Instead, Turbo will stay on the current URL upon a form submission that tries to render, rather than change it to the form action, since a reload would then issue a GET against that action URL, which may not even exist.

If the form submission is a GET request, you may render the directly rendered response by giving the form a `data-turbo-frame` target. If you'd like the URL to update as part of the rendering also pass a `data-turbo-action` attribute.

## Streaming After a Form Submission

Servers may also respond to form submissions with a [Turbo Streams](streams) message by sending the header `Content-Type: text/vnd.turbo-stream.html` followed by one or more `<turbo-stream>` elements in the response body. This lets you update multiple parts of the page without navigating.

## Prefetching Links on Hover

Turbo can also speed up perceived link navigation latency by automatically loading links on `mouseenter` events, and before the user clicks the link. This usually leads to a speed bump of 500-800ms per click navigation.

Prefetching links is enabled by default since Turbo v8, but you can disable it by adding this meta tag to your page:

```html
<meta name="turbo-prefetch" content="false">
```

To avoid prefetching links that the user is briefly hovering, Turbo waits 100ms after the user hovers over the link before prefetching it. But you may want to disable the prefetching behavior on certain links leading to pages with expensive rendering.

You can disable the behavior on a per-element basis by annotating the element or any of its ancestors with `data-turbo-prefetch="false"`.

```html
<html>
  <head>
    <meta name="turbo-prefetch" content="true">
  </head>
  <body>
    <a href="/articles">Articles</a> <!-- This link is prefetched -->
    <a href="/about" data-turbo-prefetch="false">About</a> <!-- Not prefetched -->
    <div data-turbo-prefetch="false">
      <!-- Links inside this div will not be prefetched -->
    </div>
  </body>
</html>
```

You can also disable completely the behavior on a parent and allowing on its childs one by one with `data-turbo-prefetch="true"`.

```html
<html>
  <body data-turbo-prefetch="false">
    <nav id="header" data-turbo-prefetch="true">
      <a href="/articles">Articles</a> <!-- This link is prefetched -->
      <a href="/about">About</a> <!-- This one as well -->
    </nav>
    <div id="body">
      <!-- Links inside this div will not be prefetched -->
    </div>
    <footer id="footer" data-turbo-prefetch="true">
      <!-- Links inside this footer will be prefetched -->
    </footer>
  </body>
</html>
```


You can also disable the behaviour programatically by intercepting the `turbo:before-prefetch` event and calling `event.preventDefault()`.

```javascript
document.addEventListener("turbo:before-prefetch", (event) => {
  if (isSavingData() || hasSlowInternet()) {
    event.preventDefault()
  }
})

function isSavingData() {
  return navigator.connection?.saveData
}

function hasSlowInternet() {
  return navigator.connection?.effectiveType === "slow-2g" ||
         navigator.connection?.effectiveType === "2g"
}
```

## Preload Links Into the Cache

Preload links into Turbo Drive's cache using the [data-turbo-preload][] boolean attribute.

This will make page transitions feel lightning fast by providing a preview of a page even before the first visit. Use it to preload the most important pages in your application. Avoid over usage, as it will lead to loading content that is not needed.

Not every `<a>` element can be preloaded. The `[data-turbo-preload]` attribute
won't have any effect on links that:

* navigate to another domain
* have a `[data-turbo-frame]` attribute that drives a `<turbo-frame>` element
* drive an ancestor `<turbo-frame>` element
* have the `[data-turbo="false"]` attribute
* have the `[data-turbo-stream]` attribute
* have a `[data-turbo-method]` attribute
* have an ancestor with the `[data-turbo="false"]` attribute
* have an ancestor with the `[data-turbo-prefetch="false"]` attribute

It also dovetails nicely with pages that leverage [Eager-Loading Frames](/reference/frames#eager-loaded-frame) or [Lazy-Loading Frames](/reference/frames#lazy-loaded-frame). As you can preload the structure of the page and show the user a meaningful loading state while the interesting content loads.

## Ignored Paths

Paths with a `.` in the last level of a path/URL will not be handled by Turbo unless they end in a file extension `.htm`, `.html`, `.xhtml`, or `.php`. Turbo will ignore forms and links that target these paths. The quickest way to get Turbo to target these paths is to add a `/` at the end of the URL. Examples of forms that would be ignored:

```html
<form action="/messages.67" method="post">
  <!-- ignored -->
</form>

<form action="/messages.php.1" method="post" data-turbo="true">
  <!-- also ignored -->
</form>

<form action="/messages.json" method="post" data-turbo="true">
  <!-- also ignored -->
</form>
```

The following forms would be handled:

```html
<form action="/messages/67" method="post">
  <!-- handled -->
</form>

<form action="/messages.67/action" method="post">
  <!-- also handled -->
</form>

<form action="/messages.php" method="post" data-turbo="true">
  <!-- also handled -->
</form>

<form action="/messages.json/" method="post" data-turbo="true">
  <!-- also handled -->
</form>

<form action="/messages.json/123" method="post" data-turbo="true">
  <!-- also handled -->
</form>
```

Setting any `data-turbo` methods (including `data-turbo="true"`) will not override or force Turbo to handle a path if it has a `.` that causes it to be ignored.

<br><br>

Note that preloaded `<a>` elements will dispatch [turbo:before-fetch-request](/reference/events) and [turbo:before-fetch-response](/reference/events) events. To distinguish a preloading `turbo:before-fetch-request` initiated event from an event initiated by another mechanism, check whether the request's `X-Sec-Purpose` header (read from the `event.detail.fetchOptions.headers["X-Sec-Purpose"]` property) is set to `"prefetch"`:

```js
addEventListener("turbo:before-fetch-request", (event) => {
  if (event.detail.fetchOptions.headers["X-Sec-Purpose"] === "prefetch") {
    // do additional preloading setup…
  } else {
    // do something else…
  }
})
```

[data-turbo-preload]: /reference/attributes#data-attributes
---
permalink: /handbook/page_refreshes.html
description: "Turbo can perform smooth page refreshes with morphing and scroll preservation."
---

# Smooth page refreshes with morphing

[Turbo Drive](/handbook/drive.html) makes navigation faster by avoiding full-page reloads. But there is a scenario where Turbo can raise the fidelity bar further: loading the current page again (page refresh).

A typical scenario for page refreshes is submitting a form and getting redirected back. In such scenarios, sensations significantly improve if only the changed contents get updated instead of replacing the `<body>` of the page. Turbo can do this on your behalf with morphing and scroll preservation.

${toc}

## Morphing

You can configure how Turbo handles page refresh with a `<meta name="turbo-refresh-method">` in the page's head.

```html
<head>
  ...
  <meta name="turbo-refresh-method" content="morph">
</head>
```

The possible values are `morph` or `replace` (the default). When it is `morph,` when a page refresh happens, instead of replacing the page's `<body>` contents, Turbo will only update the DOM elements that have changed, keeping the rest untouched. This approach delivers better sensations because it keeps the screen state.

Under the hood, Turbo uses the fantastic [idiomorph library](https://github.com/bigskysoftware/idiomorph).

## Scroll preservation

You can configure how Turbo handles scrolling with a `<meta name="turbo-refresh-scroll">` in the page's head.

```html
<head>
  ...
  <meta name="turbo-refresh-scroll" content="preserve">
</head>
```

The possible values are `preserve` or `reset` (the default). When it is `preserve`, when a page refresh happens, Turbo will keep the page's vertical and horizontal scroll.

## Exclude sections from morphing

Sometimes, you want to ignore certain elements while morphing. For example, you might have a popover that you want to keep open when the page refreshes. You can flag such elements with `data-turbo-permanent`, and Turbo won't attempt to morph them.

```html
<div data-turbo-permanent>...</div>
```

## Turbo frames

You can use [turbo frames](/handbook/frames.html) to define regions in your screen that will get reloaded using morphing when a page refresh happens. To do so, you must flag those frames with `refresh="morph"`.

```html
<turbo-frame id="my-frame" refresh="morph" src="/my_frame">
</turbo-frame>
```

With this mechanism, you can load additional content that didn't arrive in the initial page load (e.g., pagination). When a page refresh happens, Turbo won't remove the frame contents; instead, it will reload the turbo frame and render its contents with morphing.

## Broadcasting page refreshes

There is a new [turbo stream action](/handbook/streams.html) called `refresh` that will trigger a page refresh:

```html
<turbo-stream action="refresh"></turbo-stream>
```

Server-side frameworks can leverage these streams to offer a simple but powerful broadcasting model: the server broadcasts a single general signal, and pages smoothly refresh with morphing. 

You can see how the  [`turbo-rails`](https://github.com/hotwired/turbo-rails) gem does it for Rails:

```ruby
# In the model
class Calendar < ApplicationRecord
  broadcasts_refreshes
end

# View
turbo_stream_from @calendar
```
---
permalink: /handbook/frames.html
description: "Turbo Frames decompose pages into independent contexts, which can be lazy-loaded and scope interaction."
---

# Decompose with Turbo Frames

Turbo Frames allow predefined parts of a page to be updated on request. Any links and forms inside a frame are captured, and the frame contents automatically update after receiving a response. Regardless of whether the server provides a full document, or just a fragment containing an updated version of the requested frame, only that particular frame will be extracted from the response to replace the existing content.

Frames are created by wrapping a segment of the page in a `<turbo-frame>` element. Each element must have a unique ID, which is used to match the content being replaced when requesting new pages from the server. A single page can have multiple frames, each establishing their own context:

```html
<body>
  <div id="navigation">Links targeting the entire page</div>

  <turbo-frame id="message_1">
    <h1>My message title</h1>
    <p>My message content</p>
    <a href="/messages/1/edit">Edit this message</a>
  </turbo-frame>

  <turbo-frame id="comments">
    <div id="comment_1">One comment</div>
    <div id="comment_2">Two comments</div>

    <form action="/messages/comments">...</form>
  </turbo-frame>
</body>
```

This page has two frames: One to display the message itself, with a link to edit it. One to list all the comments, with a form to add another. Each create their own context for navigation, capturing both links and submitting forms.

When the link to edit the message is clicked, the response provided by `/messages/1/edit` has its `<turbo-frame id="message_1">` segment extracted, and the content replaces the frame from where the click originated. The edit response might look like this:

```html
<body>
  <h1>Editing message</h1>

  <turbo-frame id="message_1">
    <form action="/messages/1">
      <input name="message[name]" type="text" value="My message title">
      <textarea name="message[content]">My message content</textarea>
      <input type="submit">
    </form>
  </turbo-frame>
</body>
```

Notice how the `<h1>` isn't inside the `<turbo-frame>`. This means it will remain unchanged when the form replaces the display of the message upon editing. Only content inside a matching `<turbo-frame>` is used when the frame is updated.

Thus your page can easily play dual purposes: Make edits in place within a frame or edits outside of a frame where the entire page is dedicated to the action.

Frames serve a specific purpose: to compartmentalize the content and navigation for a fragment of the document. Their presence has ramification on any `<a>` elements or `<form>` elements contained by their child content, and shouldn't be introduced unnecessarily. Turbo Frames do not contribute support to the usage of [Turbo Stream](/handbook/streams). If your application utilizes `<turbo-frame>` elements for the sake of a `<turbo-stream>` element, change the `<turbo-frame>` into another [built-in element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element).

## Eager-Loading Frames

Frames don't have to be populated when the page that contains them is loaded. If a `src` attribute is present on the `turbo-frame` tag, the referenced URL will automatically be loaded as soon as the tag appears on the page:

```html
<body>
  <h1>Imbox</h1>

  <div id="emails">
    ...
  </div>

  <turbo-frame id="set_aside_tray" src="/emails/set_aside">
  </turbo-frame>

  <turbo-frame id="reply_later_tray" src="/emails/reply_later">
  </turbo-frame>
</body>
```

This page lists all the emails available in your <a href="http://itsnotatypo.com">imbox</a> immediately upon loading the page, but then makes two subsequent requests to present small trays at the bottom of the page for emails that have been set aside or are waiting for a later reply. These trays are created out of separate HTTP requests made to the URLs referenced in the `src`.

In the example above, the trays start empty, but it's also possible to populate the eager-loading frames with initial content, which is then overwritten when the content is fetched from the `src`:

```html
<turbo-frame id="set_aside_tray" src="/emails/set_aside">
  <img src="/icons/spinner.gif">
</turbo-frame>
```

Upon loading the imbox page, the set-aside tray is loaded from `/emails/set_aside`, and the response must contain a corresponding `<turbo-frame id="set_aside_tray">` element as in the original example:

```html
<body>
  <h1>Set Aside Emails</h1>

  <p>These are emails you've set aside</p>

  <turbo-frame id="set_aside_tray">
    <div id="emails">
      <div id="email_1">
        <a href="/emails/1">My important email</a>
      </div>
    </div>
  </turbo-frame>
</body>
```

This page now works in both its minimized form, where only the `div` with the individual emails are loaded into the tray frame on the imbox page, but also as a direct destination where a header and a description is provided. Just like in the example with the edit message form.

Note that the `<turbo-frame>` on `/emails/set_aside` does not contain a `src` attribute. That attribute is only added to the frame that needs to lazily load the content, not to the rendered frame that provides the content.

During navigation, a Frame will set `[aria-busy="true"]` on the `<turbo-frame>` element when fetching the new contents. When the navigation completes, the Frame will remove the `[aria-busy]` attribute. When navigating the `<turbo-frame>` through a `<form>` submission, Turbo will toggle the Form's `[aria-busy="true"]` attribute in tandem with the Frame's.

After navigation finishes, a Frame will set the `[complete]` attribute on the
`<turbo-frame>` element.

[aria-busy]: https://www.w3.org/TR/wai-aria/#aria-busy

## Lazy-Loading Frames

Frames that aren't visible when the page is first loaded can be marked with `loading="lazy"` such that they don't start loading until they become visible. This works exactly like the `loading="lazy"` attribute on `img`. It's a great way to delay loading of frames that sit inside `summary`/`detail` pairs or modals or anything else that starts out hidden and is then revealed.


## Cache Benefits to Loading Frames

Turning page segments into frames can help make the page simpler to implement, but an equally important reason for doing this is to improve cache dynamics. Complex pages with many segments are hard to cache efficiently, especially if they mix content shared by many with content specialized for an individual user. The more segments, the more dependent keys required for the cache look-up, the more frequently the cache will churn.

Frames are ideal for separating segments that change on different timescales and for different audiences. Sometimes it makes sense to turn the per-user element of a page into a frame, if the bulk of the rest of the page is then easily shared across all users. Other times, it makes sense to do the opposite, where a heavily personalized page turns the one shared segment into a frame to serve it from a shared cache.

While the overhead of fetching loading frames is generally very low, you should still be judicious in just how many you load, especially if these frames would create load-in jitter on the page. Frames are, however, essentially free if the content isn't immediately visible upon loading the page. Either because they're hidden behind modals or below the fold.


## Targeting Navigation Into or Out of a Frame

By default, navigation within a frame will target just that frame. This is true for both following links and submitting forms. But navigation can drive the entire page instead of the enclosing frame by setting the target to `_top`. Or it can drive another named frame by setting the target to the ID of that frame.

In the example with the set-aside tray, the links within the tray point to individual emails. You don't want those links to look for frame tags that match the `set_aside_tray` ID. You want to navigate directly to that email. This is done by marking the tray frames with the `target` attribute:

```html
<body>
  <h1>Imbox</h1>
  ...
  <turbo-frame id="set_aside_tray" src="/emails/set_aside" target="_top">
  </turbo-frame>
</body>

<body>
  <h1>Set Aside Emails</h1>
  ...
  <turbo-frame id="set_aside_tray" target="_top">
    ...
  </turbo-frame>
</body>
```

Sometimes you want most links to operate within the frame context, but not others. This is also true of forms. You can add the `data-turbo-frame` attribute on non-frame elements to control this:

```html
<body>
  <turbo-frame id="message_1">
    ...
    <a href="/messages/1/edit">
      Edit this message (within the current frame)
    </a>

    <a href="/messages/1/permission" data-turbo-frame="_top">
      Change permissions (replace the whole page)
    </a>
  </turbo-frame>

  <form action="/messages/1/delete" data-turbo-frame="message_1">
    <a href="/messages/1/warning" data-turbo-frame="_self">
      Load warning within current frame
    </a>

    <input type="submit" value="Delete this message">
    (with a confirmation shown in a specific frame)
  </form>
</body>
```

## Promoting a Frame Navigation to a Page Visit

Navigating Frames provides applications with an opportunity to change part of
the page's contents while preserving the rest of the document's state (for
example, its current scroll position or focused element). There are times when
we want changes to a Frame to also affect the browser's [history][].

To promote a Frame navigation to a Visit, render the element with the
`[data-turbo-action]` attribute. The attribute supports all [Visit][] values,
and can be declared on:

* the `<turbo-frame>` element
* any `<a>` elements that navigate the `<turbo-frame>`
* any `<form>` elements that navigate the `<turbo-frame>`
* any `<input type="submit">` or `<button>` elements contained within `<form>`
  elements that navigate the `<turbo-frame>`

For example, consider a Frame that renders a paginated list of articles and
transforms navigations into ["advance" Actions][advance]:

```html
<turbo-frame id="articles" data-turbo-action="advance">
  <a href="/articles?page=2" rel="next">Next page</a>
</turbo-frame>
```

Clicking the `<a rel="next">` element will set _both_ the `<turbo-frame>`
element's `[src]` attribute _and_ the browser's path to `/articles?page=2`.

**Note:** when rendering the page after refreshing the browser, it is _the
application's_ responsibility to render the _second_ page of articles along with
any other state derived from the URL path and search parameters.

[history]: https://developer.mozilla.org/en-US/docs/Web/API/History
[Visit]: /handbook/drive#page-navigation-basics
[advance]: /handbook/drive#application-visits

## "Breaking out" from a Frame

In most cases, requests that originate from a `<turbo-frame>` are expected to fetch content for that frame (or for
another part of the page, depending on the use of the `target` or `data-turbo-frame` attributes). This means the
response should always contain the expected `<turbo-frame>` element. If a response is missing the `<turbo-frame>`
element that Turbo expects, it's considered an error; when it happens Turbo will write an informational message into the
frame, and throw an exception.

In certain, specific cases, you might want the response to a `<turbo-frame>` request to be treated as a new, full-page
navigation instead, effectively "breaking out" of the frame. The classic example of this is when a lost or expired
session causes an application to redirect to a login page. In this case, it's better for Turbo to display that login
page rather than treat it as an error.

The simplest way to achieve this is to specify that the login page requires a full-page reload, by including the
[`turbo-visit-control`][meta] meta tag:

```html
<head>
  <meta name="turbo-visit-control" content="reload">
  ...
</head>
```

If you're using Turbo Rails, you can use the `turbo_page_requires_reload` helper to accomplish the same thing.

Pages that specify `turbo-visit-control` `reload` will always result in a full-page navigation, even if the request
originated from inside a frame.

If your application needs to handle missing frames in some other way, you can intercept the
[`turbo:frame-missing`][events] event to, for example, transform the response or perform a visit to another location.

[meta]: /reference/attributes#meta-tags
[events]: /reference/events

## Anti-Forgery Support (CSRF)

Turbo provides [CSRF](https://en.wikipedia.org/wiki/Cross-site_request_forgery) protection by checking the DOM for the existence of a `<meta>` tag with a `name` value of either `csrf-param` or `csrf-token`. For example:

```html
<meta name="csrf-token" content="[your-token]">
```

Upon form submissions, the token will be automatically added to the request's headers as `X-CSRF-TOKEN`. Requests made with `data-turbo="false"` will skip adding the token to headers.

## Custom Rendering

Turbo's default `<turbo-frame>` rendering process replaces the contents of the requesting `<turbo-frame>` element with the contents of a matching `<turbo-frame>` element in the response. In practice, a `<turbo-frame>` element's contents are rendered as if they operated on by [`<turbo-stream action="update">`](/reference/streams#update) element. The underlying renderer extracts the contents of the `<turbo-frame>` in the response and uses them to replace the requesting `<turbo-frame>` element's contents. The `<turbo-frame>` element itself remains unchanged, save for the [`[src]`, `[busy]`, and `[complete]` attributes that Turbo Drive manages](/reference/frames#html-attributes) throughout the stages of the element's request-response lifecycle.

Applications can customize the `<turbo-frame>` rendering process by adding a `turbo:before-frame-render` event listener and overriding the `event.detail.render` property.

For example, you could merge the response `<turbo-frame>` element into the requesting `<turbo-frame>` element with [morphdom](https://github.com/patrick-steele-idem/morphdom):

```javascript
import morphdom from "morphdom"

addEventListener("turbo:before-frame-render", (event) => {
  event.detail.render = (currentElement, newElement) => {
    morphdom(currentElement, newElement, { childrenOnly: true })
  }
})
```

Since `turbo:before-frame-render` events bubble up the document, you can override one `<turbo-frame>` element's rendering by attaching the event listener directly to the element, or override all `<turbo-frame>` elements' rendering by attaching the listener to the `document`.

## Pausing Rendering

Applications can pause rendering and make additional preparations before continuing.

Listen for the `turbo:before-frame-render` event to be notified when rendering is about to start, and pause it using `event.preventDefault()`. Once the preparation is done continue rendering by calling `event.detail.resume()`.

An example use case is adding exit animation:

```javascript
document.addEventListener("turbo:before-frame-render", async (event) => {
  event.preventDefault()

  await animateOut()

  event.detail.resume()
})
```
---
permalink: /handbook/streams.html
description: "Turbo Streams deliver page changes over WebSocket, SSE or in response to form submissions using just HTML and a set of CRUD-like actions."
---

# Come Alive with Turbo Streams

Turbo Streams deliver page changes as fragments of HTML wrapped in `<turbo-stream>` elements. Each stream element specifies an action together with a target ID to declare what should happen to the HTML inside it. These elements can be delivered to the browser synchronously as a classic HTTP response, or asynchronously over transports such as webSockets, SSE, etc, to bring the application alive with updates made by other users or processes.

They can be used to surgically update the DOM after a user action such as removing an element from a list without reloading the whole page, or to implement real-time capabilities such as appending a new message to a live conversation as it is sent by a remote user.

## Stream Messages and Actions

A Turbo Streams message is a fragment of HTML consisting of `<turbo-stream>` elements. The stream message below demonstrates the nine possible stream actions:

```html
<turbo-stream action="append" target="messages">
  <template>
    <div id="message_1">
      This div will be appended to the element with the DOM ID "messages".
    </div>
  </template>
</turbo-stream>

<turbo-stream action="prepend" target="messages">
  <template>
    <div id="message_1">
      This div will be prepended to the element with the DOM ID "messages".
    </div>
  </template>
</turbo-stream>

<turbo-stream action="replace" target="message_1">
  <template>
    <div id="message_1">
      This div will replace the existing element with the DOM ID "message_1".
    </div>
  </template>
</turbo-stream>

<turbo-stream action="replace" method="morph" target="current_step">
  <template>
    <!-- The contents of this template will replace the element with ID "current_step" via morph. -->
    <li>New item</li>
  </template>
</turbo-stream>

<turbo-stream action="update" target="unread_count">
  <template>
    <!-- The contents of this template will replace the
    contents of the element with ID "unread_count" by
    setting innerHtml to "" and then switching in the
    template contents. Any handlers bound to the element
    "unread_count" would be retained. This is to be
    contrasted with the "replace" action above, where
    that action would necessitate the rebuilding of
    handlers. -->
    1
  </template>
</turbo-stream>

<turbo-stream action="update" method="morph" target="current_step">
  <template>
    <!-- The contents of this template will replace the children of the element with ID "current_step" via morph. -->
    <li>New item</li>
  </template>
</turbo-stream>

<turbo-stream action="remove" target="message_1">
  <!-- The element with DOM ID "message_1" will be removed.
  The contents of this stream element are ignored. -->
</turbo-stream>

<turbo-stream action="before" target="current_step">
  <template>
    <!-- The contents of this template will be added before the
    the element with ID "current_step". -->
    <li>New item</li>
  </template>
</turbo-stream>

<turbo-stream action="after" target="current_step">
  <template>
    <!-- The contents of this template will be added after the
    the element with ID "current_step". -->
    <li>New item</li>
  </template>
</turbo-stream>

<turbo-stream action="refresh" request-id="abcd-1234"></turbo-stream>
```

Note that every `<turbo-stream>` element must wrap its included HTML inside a `<template>` element.

A Turbo Stream can integrate with any element in the document that can be
resolved by an [id](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/id) attribute or [CSS selector](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors) (with the exception of `<template>` element or `<iframe>` element content). It is not necessary to change targeted elements into [`<turbo-frame>` elements](/handbook/frames). If your application utilizes `<turbo-frame>` elements for the sake of a `<turbo-stream>` element, change the `<turbo-frame>` into another [built-in element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element).


You can render any number of stream elements in a single stream message from a WebSocket, SSE or in response to a form submission.

Also, any `<turbo-stream>` element that's inserted into the page (e.g. through full page or frame load), will be processed by Turbo and then removed from the dom. This allows stream actions to be executed automatically when a page or frame is loaded.

## Actions With Multiple Targets

Actions can be applied against multiple targets using the `targets` attribute with a CSS query selector, instead of the regular `target` attribute that uses a dom ID reference. Examples:

```html
<turbo-stream action="remove" targets=".old_records">
  <!-- The element with the class "old_records" will be removed.
  The contents of this stream element are ignored. -->
</turbo-stream>

<turbo-stream action="after" targets="input.invalid_field">
  <template>
    <!-- The contents of this template will be added after the
    all elements that match "inputs.invalid_field". -->
    <span>Incorrect</span>
  </template>
</turbo-stream>
```

## Streaming From HTTP Responses

Turbo knows to automatically attach `<turbo-stream>` elements when they arrive in response to `<form>` submissions that declare a [MIME type][] of `text/vnd.turbo-stream.html`. When submitting a `<form>` element whose [method][] attribute is set to `POST`, `PUT`, `PATCH`, or `DELETE`, Turbo injects `text/vnd.turbo-stream.html` into the set of response formats in the request's [Accept][] header. When responding to requests containing that value in its [Accept][] header, servers can tailor their responses to deal with Turbo Streams, HTTP redirects, or other types of clients that don't support streams (such as native applications).

In a Rails controller, this would look like:

```ruby
def destroy
  @message = Message.find(params[:id])
  @message.destroy

  respond_to do |format|
    format.turbo_stream { render turbo_stream: turbo_stream.remove(@message) }
    format.html         { redirect_to messages_url }
  end
end
```

By default, Turbo doesn't add the `text/vnd.turbo-stream.html` MIME type when submitting links, or forms with a method type of `GET`. To use Turbo Streams responses with `GET` requests in an application you can instruct Turbo to include the MIME type by adding a `data-turbo-stream` attribute to a link or form.

[MIME type]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types/Common_types
[method]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/form#attr-method
[Accept]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept

## Reusing Server-Side Templates

The key to Turbo Streams is the ability to reuse your existing server-side templates to perform live, partial page changes. The HTML template used to render each message in a list of such on the first page load is the same template that'll be used to add one new message to the list dynamically later. This is at the essence of the HTML-over-the-wire approach: You don't need to serialize the new message as JSON, receive it in JavaScript, render a client-side template. It's just the standard server-side templates reused.

Another example from how this would look in Rails:

```erb
<!-- app/views/messages/_message.html.erb -->
<div id="<%= dom_id message %>">
  <%= message.content %>
</div>

<!-- app/views/messages/index.html.erb -->
<h1>All the messages</h1>
<%= render partial: "messages/message", collection: @messages %>
```

```ruby
# app/controllers/messages_controller.rb
class MessagesController < ApplicationController
  def index
    @messages = Message.all
  end

  def create
    message = Message.create!(params.require(:message).permit(:content))

    respond_to do |format|
      format.turbo_stream do
        render turbo_stream: turbo_stream.append(:messages, partial: "messages/message",
          locals: { message: message })
      end

      format.html { redirect_to messages_url }
    end
  end
end
```

When the form to create a new message submits to the `MessagesController#create` action, the very same partial template that was used to render the list of messages in `MessagesController#index` is used to render the turbo-stream action. This will come across as a response that looks like this:

```html
Content-Type: text/vnd.turbo-stream.html; charset=utf-8

<turbo-stream action="append" target="messages">
  <template>
    <div id="message_1">
      The content of the message.
    </div>
  </template>
</turbo-stream>
```

This `messages/message` template partial can then also be used to re-render the message following an edit/update operation. Or to supply new messages created by other users over a WebSocket or a SSE connection. Being able to reuse the same templates across the whole spectrum of use is incredibly powerful, and key to reducing the amount of work it takes to create these modern, fast applications.

## Progressively Enhance When Necessary

It's good practice to start your interaction design without Turbo Streams. Make the entire application work as it would if Turbo Streams were not available, then layer them on as a level-up. This means you won't come to rely on the updates for flows that need to work in native applications or elsewhere without them.

The same is especially true for WebSocket updates. On poor connections, or if there are server issues, your WebSocket may well get disconnected. If the application is designed to work without it, it'll be more resilient.

## But What About Running JavaScript?

Turbo Streams consciously restricts you to nine actions: append, prepend, (insert) before, (insert) after, replace, update, remove, morph, and refresh. If you want to trigger additional behavior when these actions are carried out, you should attach behavior using <a href="https://stimulus.hotwired.dev">Stimulus</a> controllers. This restriction allows Turbo Streams to focus on the essential task of delivering HTML over the wire, leaving additional logic to live in dedicated JavaScript files.

Embracing these constraints will keep you from turning individual responses into a jumble of behaviors that cannot be reused and which make the app hard to follow. The key benefit from Turbo Streams is the ability to reuse templates for initial rendering of a page through all subsequent updates.

## Custom Actions

By default, Turbo Streams support [nine values for its `action` attribute](/reference/streams#the-seven-actions). If your application needs to support other behaviors, you can override the `event.detail.render` function.

For example, if you'd like to expand upon the nine actions to support `<turbo-stream>` elements with `[action="alert"]` or `[action="log"]`, you could declare a `turbo:before-stream-render` listener to provide custom behavior:

```javascript
addEventListener("turbo:before-stream-render", ((event) => {
  const fallbackToDefaultActions = event.detail.render

  event.detail.render = function (streamElement) {
    if (streamElement.action == "alert") {
      // ...
    } else if (streamElement.action == "log") {
      // ...
    } else {
      fallbackToDefaultActions(streamElement)
    }
  }
}))
```

In addition to listening for `turbo:before-stream-render` events, applications
can also declare actions as properties directly on `StreamActions`:

```javascript
import { StreamActions } from "@hotwired/turbo"

// <turbo-stream action="log" message="Hello, world"></turbo-stream>
//
StreamActions.log = function () {
  console.log(this.getAttribute("message"))
}
```

## Integration with Server-Side Frameworks

Of all the techniques that are included with Turbo, it's with Turbo Streams you'll see the biggest advantage from close integration with your backend framework. As part of the official Hotwire suite, we've created a reference implementation for what such an integration can look like in the <a href="https://github.com/hotwired/turbo-rails">turbo-rails gem</a>. This gem relies on the built-in support for both WebSockets and asynchronous rendering present in Rails through the Action Cable and Active Job frameworks, respectively.

Using the <a href="https://github.com/hotwired/turbo-rails/blob/main/app/models/concerns/turbo/broadcastable.rb">Broadcastable</a> concern mixed into Active Record, you can trigger WebSocket updates directly from your domain model. And using the <a href="https://github.com/hotwired/turbo-rails/blob/main/app/models/turbo/streams/tag_builder.rb">Turbo::Streams::TagBuilder</a>, you can render `<turbo-stream>` elements in inline controller responses or dedicated templates, invoking the eight actions with associated rendering through a simple DSL.

Turbo itself is completely backend-agnostic, though. So we encourage other frameworks in other ecosystems to look at the reference implementation provided for Rails to create their own tight integration.

Turbo's `<turbo-stream-source>` custom element connects to a stream source
through its `[src]` attribute. When declared with an `ws://` or `wss://` URL,
the underlying stream source will be a [WebSocket][] instance. Otherwise, the
connection is through an [EventSource][].

When the element is connected to the document, the stream source is
connected. When the element is disconnected, the stream is disconnected.

Since the document's `<head>` is persistent across Turbo navigations, it's
important to mount the `<turbo-stream-source>` as a descendant of the document's
`<body>` element.

Typical full page navigations driven by Turbo will result in the `<body>` contents
being discarded and replaced with the resulting document. It's the server's
responsibility to ensure that the element is present on any page that requires
streaming.

Alternatively, a straightforward way to integrate any backend application with Turbo Streams is to rely on [the Mercure protocol](https://mercure.rocks). Mercure defines a convenient way for server applications to broadcast page changes to every connected clients through [Server-Sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events). [Learn how to use Mercure with Turbo Streams](https://mercure.rocks/docs/ecosystem/hotwire).

[WebSocket]: https://developer.mozilla.org/en-US/docs/Web/API/WebSocket
[EventSource]: https://developer.mozilla.org/en-US/docs/Web/API/EventSource
---
permalink: /handbook/native.html
description: "Hotwire Native lets your majestic monolith form the center of your native iOS and Android apps, with seamless transitions between web and native sections."
---

# Go Native on iOS & Android

Hotwire Native for iOS provides the tooling to wrap your Turbo-enabled web app in a native iOS shell. It manages a single WKWebView instance across multiple view controllers, giving you native navigation UI with all the client-side performance benefits of Turbo. See <a href="https://github.com/hotwired/hotwire-native-ios/">Hotwire Native: iOS</a> for more details.

Hotwire Native for Android provides the same kind of tooling, managing a single WebView instance across multiple Fragment destinations. See <a href="https://github.com/hotwired/hotwire-native-android/">Hotwire Native: Android</a> for more details.

The best way to see what's possible with the native adapters is to setup the demo native application. We have one [for iOS](https://native.hotwired.dev/ios/getting-started) and [for Android](https://native.hotwired.dev/android/getting-started). You can open the code in your native environments and follow along to explore all the features.
---
permalink: /handbook/building.html
description: "Learn more about building an application with Turbo."
---

# Building Your Turbo Application

Turbo is fast because it prevents the whole page from reloading when you follow a link or submit a form. Your application becomes a persistent, long-running process in the browser. This requires you to rethink the way you structure your JavaScript.

In particular, you can no longer depend on a full page load to reset your environment every time you navigate. The JavaScript `window` and `document` objects retain their state across page changes, and any other objects you leave in memory will stay in memory.

With awareness and a little extra care, you can design your application to gracefully handle this constraint without tightly coupling it to Turbo.

## Working with Script Elements

Your browser automatically loads and evaluates any `<script>` elements present on the initial page load.

When you navigate to a new page, Turbo Drive looks for any `<script>` elements in the new page’s `<head>` which aren’t present on the current page. Then it appends them to the current `<head>` where they’re loaded and evaluated by the browser. You can use this to load additional JavaScript files on-demand.

Turbo Drive evaluates `<script>` elements in a page’s `<body>` each time it renders the page. You can use inline body scripts to set up per-page JavaScript state or bootstrap client-side models. To install behavior, or to perform more complex operations when the page changes, avoid script elements and use the `turbo:load` event instead.

Annotate `<script>` elements with `data-turbo-eval="false"` if you do not want Turbo to evaluate them after rendering. Note that this annotation will not prevent your browser from evaluating scripts on the initial page load.

### Loading Your Application’s JavaScript Bundle

Always make sure to load your application’s JavaScript bundle using `<script>` elements in the `<head>` of your document. Otherwise, Turbo Drive will reload the bundle with every page change.

```html
<head>
  ...
  <script src="/application-cbd3cd4.js" defer></script>
</head>
```

You should also consider configuring your asset packaging system to fingerprint each script so it has a new URL when its contents change. Then you can use the `data-turbo-track` attribute to force a full page reload when you deploy a new JavaScript bundle. See [Reloading When Assets Change](/handbook/drive#reloading-when-assets-change) for information.

## Understanding Caching

Turbo Drive maintains a cache of recently visited pages. This cache serves two purposes: to display pages without accessing the network during restoration visits, and to improve perceived performance by showing temporary previews during application visits.

When navigating by history (via [Restoration Visits](/handbook/drive#restoration-visits)), Turbo Drive will restore the page from cache without loading a fresh copy from the network, if possible.

Otherwise, during standard navigation (via [Application Visits](/handbook/drive#application-visits)), Turbo Drive will immediately restore the page from cache and display it as a preview while simultaneously loading a fresh copy from the network. This gives the illusion of instantaneous page loads for frequently accessed locations.

Turbo Drive saves a copy of the current page to its cache just before rendering a new page. Note that Turbo Drive copies the page using [`cloneNode(true)`](https://developer.mozilla.org/en-US/docs/Web/API/Node/cloneNode), which means any attached event listeners and associated data are discarded.

### Preparing the Page to be Cached

Listen for the `turbo:before-cache` event if you need to prepare the document before Turbo Drive caches it. You can use this event to reset forms, collapse expanded UI elements, or tear down any third-party widgets so the page is ready to be displayed again.

```js
document.addEventListener("turbo:before-cache", function() {
  // ...
})
```

Certain page elements are inherently _temporary_, like flash messages or alerts. If they’re cached with the document they’ll be redisplayed when it’s restored, which is rarely desirable. You can annotate such elements with `data-turbo-temporary` to have Turbo Drive automatically remove them from the page before it’s cached.

```html
<body>
  <div class="flash" data-turbo-temporary>
    Your cart was updated!
  </div>
  ...
</body>
```

### Detecting When a Preview is Visible

Turbo Drive adds a `data-turbo-preview` attribute to the `<html>` element when it displays a preview from cache. You can check for the presence of this attribute to selectively enable or disable behavior when a preview is visible.

```js
if (document.documentElement.hasAttribute("data-turbo-preview")) {
  // Turbo Drive is displaying a preview
}
```

### Opting Out of Caching

You can control caching behavior on a per-page basis by including a `<meta name="turbo-cache-control">` element in your page’s `<head>` and declaring a caching directive.

Use the `no-preview` directive to specify that a cached version of the page should not be shown as a preview during an application visit. Pages marked no-preview will only be used for restoration visits.

To specify that a page should not be cached at all, use the `no-cache` directive. Pages marked no-cache will always be fetched over the network, including during restoration visits.

```html
<head>
  ...
  <meta name="turbo-cache-control" content="no-cache">
</head>
```

To completely disable caching in your application, ensure every page contains a no-cache directive.

### Opting Out of Caching from the client-side

The value of the `<meta name="turbo-cache-control">` element can also be controlled by a client-side API exposed via `Turbo.cache`.

```js
// Set cache control of current page to `no-cache`
Turbo.cache.exemptPageFromCache()

// Set cache control of current page to `no-preview`
Turbo.cache.exemptPageFromPreview()
```

Both functions will create a `<meta name="turbo-cache-control">` element in the `<head>` if the element is not already present.

A previously set cache control value can be reset via:

```js
Turbo.cache.resetCacheControl()
```

## Installing JavaScript Behavior

You may be used to installing JavaScript behavior in response to the `window.onload`, `DOMContentLoaded`, or jQuery `ready` events. With Turbo, these events will fire only in response to the initial page load, not after any subsequent page changes. We compare two strategies for connecting JavaScript behavior to the DOM below.

### Observing Navigation Events

Turbo Drive triggers a series of events during navigation. The most significant of these is the `turbo:load` event, which fires once on the initial page load, and again after every Turbo Drive visit.

You can observe the `turbo:load` event in place of `DOMContentLoaded` to set up JavaScript behavior after every page change:

```js
document.addEventListener("turbo:load", function() {
  // ...
})
```

Keep in mind that your application will not always be in a pristine state when this event is fired, and you may need to clean up behavior installed for the previous page.

Also note that Turbo Drive navigation may not be the only source of page updates in your application, so you may wish to move your initialization code into a separate function which you can call from `turbo:load` and anywhere else you may change the DOM.

When possible, avoid using the `turbo:load` event to add other event listeners directly to elements on the page body. Instead, consider using [event delegation](https://learn.jquery.com/events/event-delegation/) to register event listeners once on `document` or `window`.

See the [Full List of Events](/reference/events) for more information.

### Attaching Behavior With Stimulus

New DOM elements can appear on the page at any time by way of frame navigation, stream messages, or client-side rendering operations, and these elements often need to be initialized as if they came from a fresh page load.

You can handle all of these updates, including updates from Turbo Drive page loads, in a single place with the conventions and lifecycle callbacks provided by Turbo's sister framework, [Stimulus](https://stimulus.hotwired.dev).

Stimulus lets you annotate your HTML with controller, action, and target attributes:

```html
<div data-controller="hello">
  <input data-hello-target="name" type="text">
  <button data-action="click->hello#greet">Greet</button>
</div>
```

Implement a compatible controller and Stimulus connects it automatically:

```js
// hello_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  greet() {
    console.log(`Hello, ${this.name}!`)
  }

  get name() {
    return this.targets.find("name").value
  }
}
```

Stimulus connects and disconnects these controllers and their associated event handlers whenever the document changes using the [MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) API. As a result, it handles Turbo Drive page changes, Turbo Frames navigation, and Turbo Streams messages the same way it handles any other type of DOM update.

## Making Transformations Idempotent

Often you’ll want to perform client-side transformations to HTML received from the server. For example, you might want to use the browser’s knowledge of the user’s current time zone to group a collection of elements by date.

Suppose you have annotated a set of elements with `data-timestamp` attributes indicating the elements’ creation times in UTC. You have a JavaScript function that queries the document for all such elements, converts the timestamps to local time, and inserts date headers before each element that occurs on a new day.

Consider what happens if you’ve configured this function to run on `turbo:load`. When you navigate to the page, your function inserts date headers. Navigate away, and Turbo Drive saves a copy of the transformed page to its cache. Now press the Back button—Turbo Drive restores the page, fires `turbo:load` again, and your function inserts a second set of date headers.

To avoid this problem, make your transformation function _idempotent_. An idempotent transformation is safe to apply multiple times without changing the result beyond its initial application.

One technique for making a transformation idempotent is to keep track of whether you’ve already performed it by setting a `data` attribute on each processed element. When Turbo Drive restores your page from cache, these attributes will still be present. Detect these attributes in your transformation function to determine which elements have already been processed.

A more robust technique is simply to detect the transformation itself. In the date grouping example above, that means checking for the presence of a date divider before inserting a new one. This approach gracefully handles newly inserted elements that weren’t processed by the original transformation.

## Persisting Elements Across Page Loads

Turbo Drive allows you to mark certain elements as _permanent_. Permanent elements persist across page loads, so that any changes you make to those elements do not need to be reapplied after navigation.

Consider a Turbo Drive application with a shopping cart. At the top of each page is an icon with the number of items currently in the cart. This counter is updated dynamically with JavaScript as items are added and removed.

Now imagine a user who has navigated to several pages in this application. She adds an item to her cart, then presses the Back button in her browser. Upon navigation, Turbo Drive restores the previous page’s state from cache, and the cart item count erroneously changes from 1 to 0.

You can avoid this problem by marking the counter element as permanent. Designate permanent elements by giving them an HTML `id` and annotating them with `data-turbo-permanent`.

```html
<div id="cart-counter" data-turbo-permanent>1 item</div>
```

Before each render, Turbo Drive matches all permanent elements by ID and transfers them from the original page to the new page, preserving their data and event listeners.
---
permalink: /handbook/installing.html
description: "Learn how to install Turbo in your application."
---

# Installing Turbo in Your Application

Turbo can either be referenced in compiled form via the Turbo distributable script directly in the `<head>` of your application or through npm via a bundler like esbuild.

## In Compiled Form

You can float on the latest release of Turbo using a CDN bundler like jsDelivr. Just include a `<script>` tag in the `<head>` of your application:

```html
<head>
  <script type="module" src="https://cdn.jsdelivr.net/npm/@hotwired/turbo@latest/dist/turbo.es2017-esm.min.js"></script>
</head>
```

Or <a href="https://unpkg.com/browse/@hotwired/turbo@latest/dist/">download the compiled packages from unpkg</a>.

## As An npm Package

You can install Turbo from npm via the `npm` or `yarn` packaging tools. 

If you using any Turbo functions such as `Turbo.visit()` import the `Turbo` functions into your code:

```javascript
import * as Turbo from "@hotwired/turbo"
```

If you're *not* using any Turbo functions such as `Turbo.visit()` import the library. This avoids issues with tree-shaking and unused variables in some bundlers. See [Import a module for its side effects only](https://developer.mozilla.org/en-US/docs/web/javascript/reference/statements/import#import_a_module_for_its_side_effects_only) on MDN.

```javascript
import "@hotwired/turbo";
```

## In a Ruby on Rails application

The Turbo JavaScript framework is included with [the turbo-rails gem](https://github.com/hotwired/turbo-rails) for direct use with the asset pipeline.

# Reference
---
permalink: /reference/attributes.html
order: 05
title: "Attributes"
description: "A reference of everything you can do with element attributes and meta tags."
---

# Attributes and Meta Tags

## Data Attributes

The following data attributes can be applied to elements to customize Turbo's behaviour.

* `data-turbo="false"` disables Turbo Drive on links and forms including descendants. To reenable when an ancestor has opted out, use `data-turbo="true"`. Be careful: when Turbo Drive is disabled, browsers treat link clicks as normal, but [native adapters](/handbook/native) may exit the app. Note that if Turbo [ignores a path](/handbook/drive#ignored-paths), then setting `data-turbo="true"` will not force/override it to enable.
* `data-turbo-track="reload"` tracks the element's HTML and performs a full page reload when it changes. Typically used to [keep `script` and CSS `link` elements up-to-date](/handbook/drive#reloading-when-assets-change).
* `data-turbo-track="dynamic"` tracks the element's HTML and removes the element when it is absent from an HTML response. Typically used to [remove `style` and `link` elements](/handbook/drive#removing-assets-when-they-change) during navigation.
* `data-turbo-frame` identifies the Turbo Frame to navigate. Refer to the [Frames documentation](/reference/frames) for further details.
* `data-turbo-preload` signals to [Drive](/handbook/drive#preload-links-into-the-cache) to pre-fetch the next page's content
* `data-turbo-prefetch="false"` [disables prefetching links](/handbook/drive#prefetching-links-on-hover) when the element is hovered.
* `data-turbo-action` customizes the [Visit](/handbook/drive#page-navigation-basics) action. Valid values are `replace` or `advance`. Can also be used with Turbo Frames to [promote frame navigations to page visits](/handbook/frames#promoting-a-frame-navigation-to-a-page-visit).
* `data-turbo-permanent` [persists the element between page loads](/handbook/building#persisting-elements-across-page-loads). The element must have a unique `id` attribute. It also serves to exclude elements from being morphed when using [page refreshes with morphing](/handbook/page_refreshes.html)
* `data-turbo-temporary` removes the element before the document is cached, preventing it from reappearing when restored.
* `data-turbo-eval="false"` prevents inline `script` elements from being re-evaluated on Visits.
* `data-turbo-method` changes the link request type from the default `GET`. Ideally, non-`GET` requests should be triggered with forms, but `data-turbo-method` might be useful where a form is not possible.
* `data-turbo-stream` specifies that a link or form can accept a Turbo Streams response. Turbo [automatically requests stream responses](/handbook/streams#streaming-from-http-responses) for form submissions with non-`GET` methods; `data-turbo-stream` allows Turbo Streams to be used with `GET` requests as well.
* `data-turbo-confirm` presents a confirm dialog with the given value. Can be used on `form` elements or links with `data-turbo-method`.
* `data-turbo-submits-with` specifies text to display when submitting a form. Can be used on `input` or `button` elements. While the form is submitting the text of the element will show the value of `data-turbo-submits-with`. After the submission, the original text will be restored. Useful for giving user feedback by showing a message like "Saving..." while an operation is in progress.
* `data-turbo-prefetch="false"` disables [prefetching links on hover](/handbook/drive#prefetching-links-on-hover)
* `download` opts out of Turbo since the link is for downloading files and not navigation.

## Automatically Added Attributes

The following attributes are automatically added by Turbo and are useful to determine the Visit state at a given moment.

* `disabled` is added to the form submitter while the form request is in progress, to prevent repeat submissions.
* `data-turbo-preview` is added to the `html` element when displaying a [preview](/handbook/building#detecting-when-a-preview-is-visible) during a Visit.
* `data-turbo-visit-direction` is added to the `html` element during a visit, with a value of `forward` or `back` or `none`, to indicate its direction.
* `aria-busy` is added to:
  * the `html` element when a visit is in progress
  * the `form` element when a submission is in progress.
  * the `turbo-frame` element when a visit or form submission is in progress within the frame.
* `busy` is added to the `turbo-frame` element when a navigation or form submission is in progress within the frame.

## Meta Tags

The following `meta` elements, added to the `head`, can be used to customize caching and Visit behavior.

* `<meta name="turbo-cache-control">` to [opt out of caching](/handbook/building#opting-out-of-caching).
* `<meta name="turbo-visit-control" content="reload">` will perform a full page reload whenever Turbo navigates to the page, including when the request originates from a `<turbo-frame>`.
* `<meta name="turbo-root">` to [scope Turbo Drive to a particular root location](/handbook/drive#setting-a-root-location).
* `<meta name="view-transition" content="same-origin">` to trigger view transitions on browsers that support the [View Transition API](https://caniuse.com/view-transitions).
* `<meta name="turbo-refresh-method" content="morph">` will configure [page refreshes with morphing](/handbook/page_refreshes.html).
* `<meta name="turbo-refresh-scroll" content="preserve">` will enable [scroll preservation during page refreshes](/handbook/page_refreshes.html).
* `<meta name="turbo-prefetch" content="false">` will disable [prefetching links on hover](/handbook/drive#prefetching-links-on-hover).
---
permalink: /reference/drive.html
order: 01
description: "A reference of everything you can do with Turbo Drive."
---

# Drive

## Turbo.visit

```js
Turbo.visit(location)
Turbo.visit(location, { action: action })
Turbo.visit(location, { frame: frame })
```

Performs an [Application Visit][] to the given _location_ (a string containing a URL or path) with the specified _action_ (a string, either `"advance"` or `"replace"`).

If _location_ is a cross-origin URL, or falls outside of the specified root (see [Setting a Root Location](/handbook/drive#setting-a-root-location)), Turbo performs a full page load by setting `window.location`.

If _action_ is unspecified, Turbo Drive assumes a value of `"advance"`.

Before performing the visit, Turbo Drive fires a `turbo:before-visit` event on `document`. Your application can listen for this event and cancel the visit with `event.preventDefault()` (see [Canceling Visits Before They Start](/handbook/drive#canceling-visits-before-they-start)).

If _frame_ is specified, find a `<turbo-frame>` element with an `[id]` attribute that matches the provided value, and navigate it to the provided _location_. If the `<turbo-frame>` cannot be found, perform a page-level [Application Visit][].

[Application Visit]: /handbook/drive#application-visits

## Turbo.cache.clear

```js
Turbo.cache.clear()
```

Removes all entries from the Turbo Drive page cache. Call this when state has changed on the server that may affect cached pages.

**Note:** This function was previously exposed as `Turbo.clearCache()`. The top-level function was deprecated in favor of the new `Turbo.cache.clear()` function.

## Turbo.config.drive.progressBarDelay

```js
Turbo.config.drive.progressBarDelay = delayInMilliseconds
```

Sets the delay after which the [progress bar](/handbook/drive#displaying-progress) will appear during navigation, in milliseconds. The progress bar appears after 500ms by default.

Note that this method has no effect when used with the iOS or Android adapters.

**Note:** This function was previously exposed as `Turbo.setProgressBarDelay` function. The top-level function was deprecated in favor of the new `Turbo.config.drive.progressBarDelay = delayInMilliseconds` syntax.

## Turbo.config.forms.confirm

```js
Turbo.config.forms.confirm = confirmMethod
```

Sets the method that is called by links decorated with [`data-turbo-confirm`](/handbook/drive#requiring-confirmation-for-a-visit). The default is the browser's built in `confirm`. The method should return a `Promise` object that resolves to true or false, depending on whether the visit should proceed.

**Note:** This function was previously exposed as `Turbo.setConfirmMethod` function. The top-level function was deprecated in favor of the new `Turbo.config.forms.confirm = confirmMethod` syntax.

## Turbo.session.drive

```js
Turbo.session.drive = false
```

Turns Turbo Drive off by default. You must now opt-in to Turbo Drive on a per-link and per-form basis using `data-turbo="true"`.

## `FetchRequest`

Turbo dispatches a variety of [events while making HTTP requests](/reference/events#http-requests) that reference `FetchRequest` objects with the following properties:

| Property          | Type                                                                              | Description
|-------------------|-----------------------------------------------------------------------------------|------------
| `body`            | [FormData][] \| [URLSearchParams][]                                               | a [URLSearchParams][] instance for `"get"` requests, [FormData][] otherwise
| `enctype`         | `"application/x-www-form-urlencoded" \| "multipart/form-data" \| "text/plain"`    | the [HTMLFormElement.enctype][] value
| `fetchOptions`    | [RequestInit][]                                                                   | the request's configuration options
| `headers`         | [Headers][] \| `{ [string]: [any] }`                                              | the request's HTTP headers
| `method`          | `"get" \| "post" \| "put" \| "patch" \| "delete"`                                 | the HTTP verb
| `params`          | [URLSearchParams][]                                                               | the [URLSearchParams][] instance for `"get"` requests
| `target`          | [HTMLFormElement][] \| [HTMLAnchorElement][] \| `FrameElement` \| `null`          | the element responsible for initiating the request
| `url`             | [URL][]                                                                           | the request's [URL][]

[HTMLAnchorElement]: https://developer.mozilla.org/en-US/docs/Web/API/HTMLAnchorElement
[RequestInit]: https://developer.mozilla.org/en-US/docs/Web/API/Request/Request#options
[Headers]: https://developer.mozilla.org/en-US/docs/Web/API/Request/Request#headers
[HTMLFormElement.enctype]: https://developer.mozilla.org/en-US/docs/Web/API/HTMLFormElement/enctype

## `FetchResponse`

Turbo dispatches a variety of [events while making HTTP requests](/reference/events#http-requests) that reference `FetchResponse` objects with the following properties:

| Property          | Type               | Description
|-------------------|--------------------|------------
| `clientError`     | `boolean`          | `true` if the status is between 400 and 499, `false` otherwise
| `contentType`     | `string` \| `null` | the value of the [Content-Type][] header
| `failed`          | `boolean`          | `true` if the response did not succeed, `false` otherwise
| `isHTML`          | `boolean`          | `true` if the content type is HTML, `false` otherwise
| `location`        | [URL][]            | the value of [Response.url][]
| `redirected`      | `boolean`          | the value of [Response.redirected][]
| `responseHTML`    | `Promise<string>`  | clones the `Response` if its HTML, then calls [Response.text()][]
| `responseText`    | `Promise<string>`  | clones the `Response`, then calls [Response.text()][]
| `response`        | [Response]         | the `Response` instance
| `serverError`     | `boolean`          | `true` if the status is between 500 and 599, `false` otherwise
| `statusCode`      | `number`           | the value of [Response.status][]
| `succeeded`       | `boolean`          | `true` if the [Response.ok][], `false` otherwise

[Response]: https://developer.mozilla.org/en-US/docs/Web/API/Response
[Response.url]: https://developer.mozilla.org/en-US/docs/Web/API/Response/url
[Response.ok]: https://developer.mozilla.org/en-US/docs/Web/API/Response/ok
[Response.redirected]: https://developer.mozilla.org/en-US/docs/Web/API/Response/redirected
[Response.status]: https://developer.mozilla.org/en-US/docs/Web/API/Response/status
[Response.text]: https://developer.mozilla.org/en-US/docs/Web/API/Response/text
[Content-Type]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type

## `FormSubmission`

Turbo dispatches a variety of [events while submitting forms](/reference/events#forms) that reference `FormSubmission` objects with the following properties:

| Property          | Type                                                                             | Description
|-------------------|----------------------------------------------------------------------------------|------------
| `action`          | `string`                                                                         | where the `<form>` element is submitting to
| `body`            | [FormData][] \| [URLSearchParams][]                                              | the underlying [Request][] payload
| `enctype`         | `"application/x-www-form-urlencoded" \| "multipart/form-data" \| "text/plain"`   | the [HTMLFormElement.enctype][]
| `fetchRequest`    | [FetchRequest][]                                                                 | the underlying [FetchRequest][] instance
| `formElement`     | [HTMLFormElement][]                                                              | the `<form>` element to that is submitting
| `isSafe`          | `boolean`                                                                        | `true` if the `method` is `"get"`, `false` otherwise
| `location`        | [URL][]                                                                          | the `action` string transformed into a [URL][] instance
| `method`          | `"get" \| "post" \| "put" \| "patch" \| "delete"`                                | the HTTP verb
| `submitter`       | [HTMLButtonElement][] \| [HTMLInputElement][] \| `undefined`                     | the element responsible for submitting the `<form>` element

[FetchRequest]: #fetchrequest
[FormData]: https://developer.mozilla.org/en-US/docs/Web/API/FormData
[HTMLFormElement]: https://developer.mozilla.org/en-US/docs/Web/API/HTMLFormElement
[URLSearchParams]: https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams
[URL]: https://developer.mozilla.org/en-US/docs/Web/API/URL
---
permalink: /reference/events.html
order: 04
description: "A reference of everything you can do with Turbo Events."
---

# Events

Turbo emits a variety of [Custom Events][] types, dispatched from the following
sources:

* [Document](#document)
* [Page Refreshes](#page-refreshes)
* [Forms](#forms)
* [Frames](#frames)
* [Streams](#streams)
* [HTTP Requests](#http-requests)

When using jQuery, the data on the event must be accessed as `$event.originalEvent.detail`.

[Custom Events]: https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent

## Document

Turbo Drive emits events that allow you to track the navigation life cycle and respond to page loading. Except where noted, the following events fire on the [document.documentElement][] object (i.e., the `<html>` element).

[document.documentElement]: https://developer.mozilla.org/en-US/docs/Web/API/Document/documentElement

### `turbo:click`

Fires when you click a Turbo-enabled link. The clicked element is the [event.target][]. Access the requested location with `event.detail.url`. Cancel this event to let the click fall through to the browser as normal navigation.

| `event.detail` property   | Type              | Description
|---------------------------|-------------------|------------
| `url`                     | `string`          | the requested location
| `originalEvent`           | [MouseEvent][]    | the original [`click` event]

[`click` event]: https://developer.mozilla.org/en-US/docs/Web/API/Element/click_event
[MouseEvent]: https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent
[event.target]: https://developer.mozilla.org/en-US/docs/Web/API/Event/target

### `turbo:before-visit`

Fires before visiting a location, except when navigating by history. Access the requested location with `event.detail.url`. Cancel this event to prevent navigation.

| `event.detail` property   | Type              | Description
|---------------------------|-------------------|------------
| `url`                     | `string`          | the requested location

### `turbo:visit`

Fires immediately after a visit starts. Access the requested location with `event.detail.url` and action with `event.detail.action`.

| `event.detail` property   | Type                                  | Description
|---------------------------|---------------------------------------|------------
| `url`                     | `string`                              | the requested location
| `action`                  | `"advance" \| "replace" \| "restore"` | the visit's [Action][]

[Action]: /handbook/drive#page-navigation-basics

### `turbo:before-cache`

Fires before Turbo saves the current page to cache.

Instances of `turbo:before-cache` events do not have an `event.detail` property.

### `turbo:before-render`

Fires before rendering the page. Access the new `<body>` element with `event.detail.newBody`. Rendering can be canceled and continued with `event.detail.resume` (see [Pausing Rendering](/handbook/drive#pausing-rendering)). Customize how Turbo Drive renders the response by overriding the `event.detail.render` function (see [Custom Rendering](/handbook/drive#custom-rendering)).

| `event.detail` property   | Type                              | Description
|---------------------------|-----------------------------------|------------
| `renderMethod`            | `"replace" \| "morph"`            | the strategy that will be used to render the new content
| `newBody`                 | [HTMLBodyElement][]               | the new `<body>` element that will replace the document's current `<body>` element
| `resume`                  | `(value?: any) => void`           | called when [Pausing Requests][]
| `render`                  | `(currentBody, newBody) => void`  | override to [Customize Rendering](/handbook/drive#custom-rendering)

[HTMLBodyElement]: https://developer.mozilla.org/en-US/docs/Web/API/HTMLBodyElement
[preview]: /handbook/building#understanding-caching

### `turbo:render`

Fires after Turbo renders the page. This event fires twice during an application visit to a cached location: once after rendering the cached version, and again after rendering the fresh version.

| `event.detail` property   | Type                      | Description
|---------------------------|---------------------------|------------
| `renderMethod`            | `"replace" \| "morph"`    | the strategy used to render the new content

### `turbo:load`

Fires once after the initial page load, and again after every Turbo visit.

| `event.detail` property   | Type      | Description
|---------------------------|-----------|------------
| `url`                     | `string`  | the requested location
| `timing.visitStart`       | `number`  | timestamp at the start of the Visit
| `timing.requestStart`     | `number`  | timestamp at the start of the HTTP request for the next page
| `timing.requestEnd`       | `number`  | timestamp at the end of the HTTP request for the next page
| `timing.visitEnd`         | `number`  | timestamp at the end of the Visit


## Page Refreshes

Turbo Drive emits events while morphing the page's content.

### `turbo:morph`

Fires after Turbo morphs the page.

| `event.detail` property   | Type        | Description
|---------------------------|-------------|------------
| `currentElement`          | [Element][] | the original [Element][] that remains connected after the morph (most commonly `document.body`)
| `newElement`              | [Element][] | the [Element][] with the new attributes and children that is not connected after the morph

### `turbo:before-morph-element`

Fires before Turbo morphs an element. The [event.target][] references the original element that will remain connected to the document. Cancel this event by calling `event.preventDefault()` to skip morphing and preserve the original element, its attributes, and its children.

| `event.detail` property   | Type          | Description
|---------------------------|---------------|------------
| `newElement`              | [Element][]   | the [Element][] with the new attributes and children that is not connected after the morph

### `turbo:before-morph-attribute`

Fires before Turbo morphs an element's attributes. The [event.target][] references the original element that will remain connected to the document. Cancel this event by calling `event.preventDefault()` to skip morphing and preserve the original attribute value.

| `event.detail` property   | Type                      | Description
|---------------------------|---------------------------|------------
| `attributeName`           | `string`                  | the name of the attribute to be mutated
| `mutationType`            | `"update" \| "remove"`    | how the attribute will be mutated

### `turbo:morph-element`

Fires after Turbo morphs an element. The [event.target][] references the morphed element that remains connected after the morph.

| `event.detail` property   | Type          | Description
|---------------------------|---------------|------------
| `newElement`              | [Element][]   | the [Element][] with the new attributes and children that is not connected after the morph

[Element]: https://developer.mozilla.org/en-US/docs/Web/API/Element

## Forms

Turbo Drive emits events during submission, redirection, and submission failure. The following events fire on the `<form>` element during submission.

### `turbo:submit-start`

Fires during a form submission. Access the [FormSubmission][] object with `event.detail.formSubmission`. Abort form submission (e.g. after validation failure) with `event.detail.formSubmission.stop()`. Use `event.originalEvent.detail.formSubmission.stop()` if you're using jQuery.

| `event.detail` property   | Type                                      | Description
|---------------------------|-------------------------------------------|------------
| `formSubmission`          | [FormSubmission][]                        | the `<form>` element submission

### `turbo:submit-end`

Fires after the form submission-initiated network request completes. Access the [FormSubmission][] object with `event.detail.formSubmission` along with the properties included within `event.detail`.

| `event.detail` property   | Type                             | Description
|---------------------------|----------------------------------|------------
| `formSubmission`          | [FormSubmission][]               | the `<form>` element submission
| `success`                 | `boolean`                        | a `boolean` representing the request's success
| `fetchResponse`           | [FetchResponse][] \| `undefined` | present when a response is received, even if `success: false`. `undefined` if the request errored before a response was received
| `error`                   | [Error][] \| `undefined`         | `undefined` unless an actual fetch error occurs (e.g., network issues)

[Error]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors
[FormSubmission]: /reference/drive#formsubmission

## Frames

Turbo Frames emit events during their navigation life cycle. The following events fire on the `<turbo-frame>` element.

### `turbo:before-frame-render`

Fires before rendering the `<turbo-frame>` element. Access the new `<turbo-frame>` element with `event.detail.newFrame`. Rendering can be canceled and continued with `event.detail.resume` (see [Pausing Rendering](/handbook/frames#pausing-rendering)). Customize how Turbo Drive renders the response by overriding the `event.detail.render` function (see [Custom Rendering](/handbook/frames#custom-rendering)).

| `event.detail` property   | Type                              | Description
|---------------------------|-----------------------------------|------------
| `newFrame`                | `FrameElement`                    | the new `<turbo-frame>` element that will replace the current `<turbo-frame>` element
| `resume`                  | `(value?: any) => void`           | called when [Pausing Requests][]
| `render`                  | `(currentFrame, newFrame) => void`| override to [Customize Rendering](/handbook/drive#custom-rendering)

### `turbo:frame-render`

Fires right after a `<turbo-frame>` element renders its view. The specific `<turbo-frame>` element is the [event.target][]. Access the [FetchResponse][] object with `event.detail.fetchResponse` property.

| `event.detail` property   | Type                              | Description
|---------------------------|-----------------------------------|------------
| `fetchResponse`           | [FetchResponse][]                 | the HTTP request's response

### `turbo:frame-load`

Fires when a `<turbo-frame>` element is navigated and finishes loading (fires after `turbo:frame-render`). The specific `<turbo-frame>` element is the [event.target][].

Instances of `turbo:frame-load` events do not have an `event.detail` property.

### `turbo:frame-missing`

Fires when the response to a `<turbo-frame>` element request does not contain a matching `<turbo-frame>` element. By default, Turbo writes an informational message into the frame and throws an exception. Cancel this event to override this handling. You can access the [Response][] instance with `event.detail.response`, and perform a visit by calling `event.detail.visit(location, visitOptions)` (see [Turbo.visit][] to learn more about `VisitOptions`).

| `event.detail` property   | Type                                                                  | Description
|---------------------------|-----------------------------------------------------------------------|------------
| `response`                | [Response][]                                                          | the HTTP response for the request initiated by a `<turbo-frame>` element
| `visit`                   | `async (location: string \| URL, visitOptions: VisitOptions) => void` | a convenience function to initiate a page-wide navigation

[Response]: https://developer.mozilla.org/en-US/docs/Web/API/Response
[Turbo.visit]: /reference/drive#turbo.visit

## Streams

Turbo Streams emit events during their life cycle. The following events fire on the `<turbo-stream>` element.

### `turbo:before-stream-render`

Fires before rendering a Turbo Stream page update. Access the new `<turbo-stream>` element with `event.detail.newStream`. Customize the element's behavior by overriding the `event.detail.render` function (see [Custom Actions][]).

| `event.detail` property   | Type                              | Description
|---------------------------|-----------------------------------|------------
| `newStream`               | `StreamElement`                   | the new `<turbo-stream>` element whose action will be executed
| `render`                  | `async (currentElement) => void`  | override to define [Custom Actions][]

[Custom Actions]: /handbook/streams#custom-actions

## HTTP Requests

Turbo emits events when fetching content over HTTP. Depending on the what
initiated the request, the events could fire on:

* a `<turbo-frame>` during its navigation
* a `<form>` during its submission
* the `<html>` element during a page-wide Turbo Visit

### `turbo:before-fetch-request`

Fires before Turbo issues a network request (to fetch a page, submit a form, preload a link, etc.). Access the requested location with `event.detail.url` and the fetch options object with `event.detail.fetchOptions`. This event fires on the respective element (`<turbo-frame>` or `<form>` element) which triggers it and can be accessed with [event.target][] property. Request can be canceled and continued with `event.detail.resume` (see [Pausing Requests][]).

| `event.detail` property   | Type                              | Description
|---------------------------|-----------------------------------|------------
| `fetchOptions`            | [RequestInit][]                   | the `options` used to construct the [Request][]
| `url`                     | [URL][]                           | the request's location
| `resume`                  | `(value?: any) => void` callback  | called when [Pausing Requests][]

[RequestInit]: https://developer.mozilla.org/en-US/docs/Web/API/Request/Request#options
[Request]: https://developer.mozilla.org/en-US/docs/Web/API/Request/Request
[URL]: https://developer.mozilla.org/en-US/docs/Web/API/URL
[Pausing Requests]: /handbook/drive#pausing-requests

### `turbo:before-fetch-response`

Fires after the network request completes. Access the fetch options object with `event.detail`. This event fires on the respective element (`<turbo-frame>` or `<form>` element) which triggers it and can be accessed with [event.target][] property.

| `event.detail` property   | Type                      | Description
|---------------------------|---------------------------|------------
| `fetchResponse`           | [FetchResponse][]         | the HTTP request's response

[FetchResponse]: /reference/drive#fetchresponse

### `turbo:before-prefetch`

Fires before Turbo prefetches a link. The link is the `event.target`. Cancel this event to prevent prefetching.

### `turbo:fetch-request-error`

Fires when a form or frame fetch request fails due to network errors. This event fires on the respective element (`<turbo-frame>` or `<form>` element) which triggers it and can be accessed with [event.target][] property. This event can be canceled.

| `event.detail` property   | Type              | Description
|---------------------------|-------------------|------------
| `request`                 | [FetchRequest][]  | The HTTP request that failed
| `error`                   | [Error][]         | provides the cause of the failure

[Error]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors
[FetchRequest]: /reference/drive#fetchrequest
---
permalink: /reference/frames.html
order: 02
description: "A reference of everything you can do with Turbo Frames."
---

# Frames

## Basic frame

Confines all navigation within the frame by expecting any followed link or form submission to return a response including a matching frame tag:

```html
<turbo-frame id="messages">
  <a href="/messages/expanded">
    Show all expanded messages in this frame.
  </a>

  <form action="/messages">
    Show response from this form within this frame.
  </form>
</turbo-frame>
```

## Eager-loaded frame

Works like the basic frame, but the content is loaded from a remote `src` first.

```html
<turbo-frame id="messages" src="/messages">
  Content will be replaced when /messages has been loaded.
</turbo-frame>
```

## Lazy-loaded frame

Like an eager-loaded frame, but the content is not loaded from `src` until the frame is visible.

```html
<turbo-frame id="messages" src="/messages" loading="lazy">
  Content will be replaced when the frame becomes visible and /messages has been loaded.
</turbo-frame>
```

## Frame targeting the whole page by default

```html
<turbo-frame id="messages" target="_top">
  <a href="/messages/1">
    Following link will replace the whole page, not this frame.
  </a>

  <a href="/messages/1" data-turbo-frame="_self">
    Following link will replace just this frame.
  </a>

  <form action="/messages">
    Submitting form will replace the whole page, not this frame.
  </form>
</turbo-frame>
```

## Frame with overwritten navigation targets

```html
<turbo-frame id="messages">
  <a href="/messages/1">
    Following link will replace this frame.
  </a>

  <a href="/messages/1" data-turbo-frame="_top">
    Following link will replace the whole page, not this frame.
  </a>

  <form action="/messages" data-turbo-frame="navigation">
    Submitting form will replace the navigation frame.
  </form>
</turbo-frame>
```

## Frame that promotes navigations to Visits

```html
<turbo-frame id="messages" data-turbo-action="advance">
  <a href="/messages?page=2">Advance history to next page</a>
  <a href="/messages?page=2" data-turbo-action="replace">Replace history with next page</a>
</turbo-frame>
```

## Frame that will get reloaded with morphing during page refreshes & when they are explicitly reloaded with .reload()

```html
<turbo-frame id="my-frame" refresh="morph" src="/my_frame">
</turbo-frame>
```

# Attributes, properties, and functions

The `<turbo-frame>` element is a [custom element][] with its own set of HTML
attributes and JavaScript properties.

[custom element]: https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements

## HTML Attributes

* `src` accepts a URL or path value that controls navigation
  of the element

* `loading` has two valid [enumerated][] values: "eager" and "lazy". When
  `loading="eager"`, changes to the `src` attribute will immediately navigate
  the element. When `loading="lazy"`, changes to the `src` attribute will defer
  navigation until the element is visible in the viewport. The default value is `eager`.

* `busy` is a [boolean attribute][] toggled to be present when a
  `<turbo-frame>`-initiated request starts, and toggled false when the request
  ends

* `disabled` is a [boolean attribute][] that prevents any navigation when
  present

* `target` refers to another `<turbo-frame>` element by ID to be navigated when
  a descendant `<a>` is clicked. When `target="_top"`, navigate the window.

* `complete` is a boolean attribute whose presence or absence indicates whether
  or not the `<turbo-frame>` element has finished navigating.

* `autoscroll` is a [boolean attribute][] that controls whether or not to scroll
  a `<turbo-frame>` element (and its descendant `<turbo-frame>` elements) into
  view when after loading. Control the scroll's vertical alignment by setting the
  `data-autoscroll-block` attribute to a valid [Element.scrollIntoView({ block:
  "..." })][Element.scrollIntoView] value: one of `"end"`, `"start"`, `"center"`,
  or `"nearest"`. When `data-autoscroll-block` is absent, the default value is
  `"end"`. Control the scroll's behavior by setting the
  `data-autoscroll-behavior` attribute to a valid [Element.scrollIntoView({
    behavior:
  "..." })][Element.scrollIntoView] value: one of `"auto"`, or `"smooth"`.
  When `data-autoscroll-behavior` is absent, the default value is `"auto"`.


[boolean attribute]: https://www.w3.org/TR/html52/infrastructure.html#sec-boolean-attributes
[enumerated]: https://www.w3.org/TR/html52/infrastructure.html#keywords-and-enumerated-attributes
[Element.scrollIntoView]: https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollIntoView#parameters

## Properties

All `<turbo-frame>` elements can be controlled in JavaScript environments
through instances of the `FrameElement` class.

* `FrameElement.src` controls the pathname or URL to be loaded. Setting the `src` 
   property will immediately navigate the element. When `FrameElement.loaded` is 
   set to `"lazy"`, changes to the `src` property will defer navigation until the 
   element is visible in the viewport.

* `FrameElement.disabled` is a boolean property that controls whether or not the
  element will load

* `FrameElement.loading` controls the style (either `"eager"` or `"lazy"`) that
  the frame will loading its content.

* `FrameElement.loaded` references a [Promise][] instance that resolves once the
  `FrameElement`'s current navigation has completed.

* `FrameElement.complete` is a read-only boolean property set to `true` when the
  `FrameElement` has finished navigating and `false` otherwise.

* `FrameElement.autoscroll` controls whether or not to scroll the element into
  view once loaded

* `FrameElement.isActive` is a read-only boolean property that indicates whether
  or not the frame is loaded and ready to be interacted with

* `FrameElement.isPreview` is a read-only boolean property that returns `true`
  when the `document` that contains the element is a [preview][].

## Functions

* `FrameElement.reload()` is a function that reloads the frame element from its `src`.

[Promise]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
[preview]: https://turbo.hotwired.dev/handbook/building#detecting-when-a-preview-is-visible
---
permalink: /reference/streams.html
order: 03
description: "A reference of everything you can do with Turbo Streams."
---

# Streams

## The nine actions

### Append

Appends the content within the template tag to the container designated by the target dom id.

```html
<turbo-stream action="append" target="dom_id">
  <template>
    Content to append to container designated with the dom_id.
  </template>
</turbo-stream>
```
If the template's first element has an id that is already used by a direct child inside the container targeted by dom_id, it is replaced instead of appended.

### Prepend

Prepends the content within the template tag to the container designated by the target dom id.

```html
<turbo-stream action="prepend" target="dom_id">
  <template>
    Content to prepend to container designated with the dom_id.
  </template>
</turbo-stream>
```
If the template's first element has an id that is already used by a direct child inside the container targeted by dom_id, it is replaced instead of prepended.

### Replace

Replaces the element designated by the target dom id.

```html
<turbo-stream action="replace" target="dom_id">
  <template>
    Content to replace the element designated with the dom_id.
  </template>
</turbo-stream>
```

The `[method="morph"]` attribute can be added to the `turbo-stream` element to replace the element designated by the target dom id via morph.

```html
<turbo-stream action="replace" method="morph" target="dom_id">
  <template>
    Content to replace the element.
  </template>
</turbo-stream>
```

### Update

Updates the content within the template tag to the container designated by the target dom id.

```html
<turbo-stream action="update" target="dom_id">
  <template>
    Content to update to container designated with the dom_id.
  </template>
</turbo-stream>
```

The `[method="morph"]` attribute can be added to the `turbo-stream` element to morph only the children of the element designated by the target dom id.

```html
<turbo-stream action="update" method="morph" target="dom_id">
  <template>
    Content to replace the element.
  </template>
</turbo-stream>
```

### Remove

Removes the element designated by the target dom id.

```html
<turbo-stream action="remove" target="dom_id">
</turbo-stream>
```

### Before

Inserts the content within the template tag before the element designated by the target dom id.

```html
<turbo-stream action="before" target="dom_id">
  <template>
    Content to place before the element designated with the dom_id.
  </template>
</turbo-stream>
```

### After

Inserts the content within the template tag after the element designated by the target dom id.

```html
<turbo-stream action="after" target="dom_id">
  <template>
    Content to place after the element designated with the dom_id.
  </template>
</turbo-stream>
```

### Refresh

Initiates a [Page Refresh](/handbook/page_refreshes) to render new content with
morphing.

```html
<!-- without `[request-id]` -->
<turbo-stream action="refresh"></turbo-stream>

<!-- debounced with `[request-id]` -->
<turbo-stream action="refresh" request-id="abcd-1234"></turbo-stream>
```

## Targeting Multiple Elements

To target multiple elements with a single action, use the `targets` attribute with a CSS query selector instead of the `target` attribute.

```html
<turbo-stream action="remove" targets=".elementsWithClass">
</turbo-stream>

<turbo-stream action="after" targets=".elementsWithClass">
  <template>
    Content to place after the elements designated with the css query.
  </template>
</turbo-stream>
```

## Processing Stream Elements

Turbo can connect to any form of stream to receive and process stream actions. A stream source must dispatch [MessageEvent](https://developer.mozilla.org/en-US/docs/Web/API/MessageEvent) messages that contain the stream action HTML in the `data` attribute of that event. It's then connected by `Turbo.session.connectStreamSource(source)` and disconnected via `Turbo.session.disconnectStreamSource(source)`. If you need to process stream actions from different source than something producing `MessageEvent`s, you can use `Turbo.renderStreamMessage(streamActionHTML)` to do so.

A good way to wrap all this together is by using a custom element, like turbo-rails does with [TurboCableStreamSourceElement](https://github.com/hotwired/turbo-rails/blob/main/app/javascript/turbo/cable_stream_source_element.js).

## Stream Elements inside HTML

Turbo streams are implemented as [a custom HTML element](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements).
The element is interpreted as part of the `connectedCallback` function that the browser calls when the element is
connected to the page dom.

This means that any stream elements that are rendered into the dom will be interpreted. After being interpreted, Turbo
will remove the element from the dom. More specifically, it means that rendering stream actions inside the page or
frame content HTML will cause them to be executed. This can be used to execute additional sideffects beside the main content
loading.
