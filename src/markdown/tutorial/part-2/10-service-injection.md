<!--lint disable no-undefined-references-->

```run:server:start hidden=true cwd=super-rentals expect="Serving on http://localhost:4200/"
ember server
```

As promised, we will now work on implementing the share button!

<!-- TODO: make this a gif instead -->

![The working share button by the end of the chapter](/images/tutorial/part-2/service-injection/share-button@2x.png)

While adding the share button, you will learn about:
* Splattributes and the `class` attribute
* The router service
* Ember services vs. global variables
* Mocking services in tests

## Scoping the Feature

In order to be able to share on Twitter, we'll need to make use of the Twitter [Web Intent API](https://developer.twitter.com/en/docs/twitter-for-websites/tweet-button/guides/web-intent.html).

Conveniently, this API doesn't require us to procure any API keys; all we need to do is link to `https://twitter.com/intent/tweet`. This link will prompt the user to compose a new tweet. The API also supports us pre-populating the tweet with some text, hashtag suggestions, or even a link, all through the use of special query params.

For instance, let's say we would like to suggest a tweet with the following content:

```plain
Check out Grand Old Mansion on Super Rentals! https://super-rentals.example/rentals/grand-old-mansion
#vacation #travel #authentic #blessed #superrentals via @emberjs
```

We could open a new page to the <a href="https://twitter.com/intent/tweet?url=https%3A%2F%2Fsuper-rentals.example%2Frentals%2Fgrand-old-mansion&text=Check+out+Grand+Old+Mansion+on+Super+Rentals%21&hashtags=vacation%2Ctravel%2Cauthentic%2Cblessed%2Csuperrentals&via=emberjs" target="_blank" rel="external nofollow noopener noreferrer">following URL</a>:

```plain
https://twitter.com/intent/tweet?
  url=https%3A%2F%2Fsuper-rentals.example%2Frentals%2Fgrand-old-mansion&
  text=Check+out+Grand+Old+Mansion+on+Super+Rentals%21&
  hashtags=vacation%2Ctravel%2Cauthentic%2Cblessed%2Csuperrentals&
  via=emberjs
```

Of course, the user will still have the ability to edit the tweet, or they can decide to just not tweet it at all.

For our app, it probably makes the most sense for our share button to automatically share the current page's URL.

## Splattributes and the `class` Attribute

Now that we have a better understanding of the scope of this feature, let's get to work and generate a `share-button` component.

```run:command cwd=super-rentals
ember generate component share-button --with-component-class
```

```run:command hidden=true cwd=super-rentals
ember test --path dist
git add app/components/share-button.hbs
git add app/components/share-button.js
git add tests/integration/components/share-button-test.js
```

Let's start with the template that was generated for this component. We already have some markup for the share button in the `<Rental::Detailed>` component we made earlier, so let's just copy that over into our new `<ShareButton>` component.

```run:file:patch lang=handlebars cwd=super-rentals filename=app/components/share-button.hbs
@@ -1 +1,9 @@
-{{yield}}
\ No newline at end of file
+<a
+  ...attributes
+  href={{this.shareURL}}
+  target="_blank"
+  rel="external nofollow noopener noreferrer"
+  class="share button"
+>
+  {{yield}}
+</a>
```

Notice that we added `...attributes` to our `<a>` tag here. As [we learned earlier](../../part-1/reusable-components/) when working on our `<Map>` component, the order of `...attributes` relative to other attributes is significant. We don't want to allow `href`, `target`, or `rel` to be overridden by the invoker, so we specified those attributes after `...attributes`.

But what happens to the `class` attribute? Well, as it turns out, the `class` attribute is the one exception to how these component attributes are overridden! While all other HTML attributes follow the "last-write wins" rule, the values for the `class` attribute are merged together (concatenated) instead. There is a good reason for this: it allows the component to specify whatever classes that *it* needs, while allowing the invokers of the component to freely add any extra classes that *they* need for styling purposes.

We also have a `{{yield}}` inside of our `<a>` tag so that we can customize the text for the link later when invoking the `<ShareButton>` component.

## Accessing the Current Page URL

Whew! Let's look at the JavaScript class next.

```run:file:patch lang=js cwd=super-rentals filename=app/components/share-button.js
@@ -3 +3,27 @@
-export default class ShareButtonComponent extends Component {}
+const TWEET_INTENT = 'https://twitter.com/intent/tweet';
+
+export default class ShareButtonComponent extends Component {
+  get currentURL() {
+    return window.location.href;
+  }
+
+  get shareURL() {
+    let url = new URL(TWEET_INTENT);
+
+    url.searchParams.set('url', this.currentURL);
+
+    if (this.args.text) {
+      url.searchParams.set('text', this.args.text);
+    }
+
+    if (this.args.hashtags) {
+      url.searchParams.set('hashtags', this.args.hashtags);
+    }
+
+    if (this.args.via) {
+      url.searchParams.set('via', this.args.via);
+    }
+
+    return url;
+  }
+}
```

The key functionality of this class is to build the appropriate URL for the Twitter Web Intent API, which is exposed to the template via the `this.shareURL` getter. It mainly involves "gluing together" the component's arguments and setting the appropriate query params on the resulting URL. Conveniently, the browser provides a handy [`URL` class](https://javascript.info/url) that handles escaping and joining of query params for us.

The other notable functionality of this class has to do with getting the current page's URL and automatically adding it to the Twitter Intent URL. To accomplish this, we defined a `currentURL` getter that simply used the browser's global [`Location` object](https://developer.mozilla.org/en-US/docs/Web/API/Window/location), which we could access at `window.location`. Among other things, it has a `href` property (`window.location.href`) that reports the current page's URL.

Let's put this component to use by invoking it from the `<Rental::Detailed>` component!

```run:file:patch lang=handlebars cwd=super-rentals filename=app/components/rental/detailed.hbs
@@ -3,5 +3,9 @@
   <p>Nice find! This looks like a nice place to stay near {{@rental.city}}.</p>
-  <a href="#" target="_blank" rel="external nofollow noopener noreferrer" class="share button">
+  <ShareButton
+    @text="Check out {{@rental.title}} on Super Rentals!"
+    @hashtags="vacation,travel,authentic,blessed,superrentals"
+    @via="emberjs"
+  >
     Share on Twitter
-  </a>
+  </ShareButton>
 </Jumbo>
```

With that, we should have a working share button!

```run:screenshot width=1024 retina=true filename=share-button.png alt="A share button that works!"
visit http://localhost:4200/rentals/grand-old-mansion?deterministic
wait  .share.button
```

> Zoey says...
>
> Feel free to try sending the tweet! However, keep in mind that your followers cannot access your local server at `http://localhost:4200/`.

```run:command hidden=true cwd=super-rentals
ember test --path dist
git add app/components/rental/detailed.hbs
git add app/components/share-button.hbs
git add app/components/share-button.js
git add tests/acceptance/super-rentals-test.js
```

## Why We Can't Test `window.location.href`

To be sure, let's add some tests! Let's start with an acceptance test:

```run:file:patch lang=js cwd=super-rentals filename=tests/acceptance/super-rentals-test.js
@@ -1,3 +1,3 @@
 import { module, test } from 'qunit';
-import { click, visit, currentURL } from '@ember/test-helpers';
+import { click, find, visit, currentURL } from '@ember/test-helpers';
 import { setupApplicationTest } from 'ember-qunit';
@@ -37,2 +37,13 @@
     assert.dom('.rental.detailed').exists();
+    assert.dom('.share.button').hasText('Share on Twitter');
+
+    let button = find('.share.button');
+
+    let tweetURL = new URL(button.href);
+    assert.strictEqual(tweetURL.host, 'twitter.com');
+
+    assert.strictEqual(
+      tweetURL.searchParams.get('url'),
+      `${window.location.origin}/rentals/grand-old-mansion`
+    );
   });
```

The main thing we want to confirm here, from an acceptance test level, is that 1) there is a share button on the page, and 2) it correctly captures the current page's URL. Ideally, we would simulate clicking on the button to confirm that it actually works. However, that would navigate us away from the test page and stop the test!

Therefore, the best we could do is to look at the `href` attribute of the link, and check that it has roughly the things we expect in there. To do that, we used the `find` test helper to find the element, and used the browser's `URL` API to parse its `href` attribute into an object that is easier to work with.

The `href` attribute contains the Twitter Intent URL, which we confirmed by checking that the `host` portion of the URL matches `twitter.com`. We could be *more* specific, such as checking that it matches `https://twitter.com/intent/tweet` exactly. However, if we include too many specific details in our acceptance test, it may fail unexpectedly in the future as the `<ShareButton>` component's implementation evolves, resulting in a *brittle* test that needs to be constantly updated. Those details are better tested in isolation with component tests, which we will add later.

The main event here is that we wanted to confirm the Twitter Intent URL includes a link to our current page's URL. We checked that by comparing its `url` query param to the expected URL, using `window.location.origin` to get the current protocol, hostname and port, which should be `http://localhost:4200`.

If we run the tests in the browser, everything should...

```run:screenshot width=1024 height=768 retina=true filename=fail.png alt="The test failed"
visit http://localhost:4200/tests?nocontainer&nolint&deterministic
wait  #qunit-banner.qunit-fail
```

...wait a minute, our tests didn't work...again!

Looking at the failure closely, the problem seems to be that the component had captured `http://localhost:4200/tests` as the "current page's URL". The issue here is that the `<ShareButton>` component uses `window.location.href` to capture the current URL. Because we are running our tests at `http://localhost:4200/tests`, that's what we got. *Technically* it's not wrong, but this is certainly not what we meant. Gotta love computers!

This brings up an interesting question – why does the `currentURL()` test helper not have the same problem? In our test, we have been writing assertions like `assert.strictEqual(currentURL(), '/about');`, and those assertions did not fail.

It turns out that this is something Ember's router handled for us. In an Ember app, the router is responsible for handling navigation and maintaining the URL. For example, when you click on a `<LinkTo>` component, it will ask the router to execute a *[route transition](../../../routing/preventing-and-retrying-transitions/)*. Normally, the router is set up to update the browser's address bar whenever it transitions into a new route. That way, your users will be able to use the browser's back button and bookmark functionality just like any other webpage.

However, during tests, the router is configured to maintain the "logical" URL internally, without updating the browser's address bar and history entries. This way, the router won't confuse the browser and its back button with hundreds of history entries as you run through your tests. The `currentURL()` taps into this piece of internal state in the router, instead of checking directly against the actual URL in the address bar using `window.location.href`.

## The Router Service

To fix our problem, we would need to do the same. Ember exposes this internal state through the *[router service](https://api.emberjs.com/ember/release/classes/RouterService)*, which we can *[inject](../../../services/#toc_accessing-services)* into our component:

```run:file:patch lang=js cwd=super-rentals filename=app/components/share-button.js
@@ -1 +1,2 @@
+import { inject as service } from '@ember/service';
 import Component from '@glimmer/component';
@@ -5,4 +6,6 @@
 export default class ShareButtonComponent extends Component {
+  @service router;
+
   get currentURL() {
-    return window.location.href;
+    return new URL(this.router.currentURL, window.location.origin);
   }
```

Here, we added the `@service router;` declaration to our component class. This injects the router service into the component, making it available to us as `this.router`. The router service has a `currentURL` property, providing the current "logical" URL as seen by Ember's router. Similar to the test helper with the same name, this is a relative URL, so we would have to join it with `window.location.origin` to get an absolute URL that we can share.

With this change, everything is now working the way we intended.

```run:command hidden=true cwd=super-rentals
ember test --path dist
git add app/components/share-button.js
git add tests/acceptance/super-rentals-test.js
```

```run:screenshot width=1024 height=960 retina=true filename=pass-1.png alt="The previously failing test is now green"
visit http://localhost:4200/tests?nocontainer&nolint&deterministic
wait  #qunit-banner.qunit-pass
```

## Ember Services vs. Global Variables

In Ember, services serve a similar role to global variables, in that they can be easily accessed by any part of your app. For example, we can inject any available service into components, as opposed to having them passed in as an argument. This allows deeply nested components to "skip through" the layers and access things that are logically global to the entire app, such as routing, authentication, user sessions, user preferences, etc. Without services, every component would have to pass through a lot of the same arguments into every component it invokes.

A major difference between services and global variables is that services are scoped to your app, instead of all the JavaScript code that is running on the same page. This allows you to have multiple scripts running on the same page without interfering with each other.

More importantly, services are designed to be easily *swappable*. In our component class, all we did was request that Ember inject the service named `router`, without specifying where that service comes from. This allows us to *replace* Ember's router service with a different object at runtime.

> Zoey says...
>
> By default, Ember infers the name of an injected service from the name of the property. If you would like the router service to be available at, say, `this.emberRouter`, you can specify `@service('router') emberRouter;` instead. `@service router;` is simply a shorthand for `@service('router') router;`.

## Mocking Services in Tests

We will take advantage of this capability in our component test:

```run:file:patch lang=js cwd=super-rentals filename=tests/integration/components/share-button-test.js
@@ -2,2 +2,3 @@
 import { setupRenderingTest } from 'ember-qunit';
+import Service from '@ember/service';
 import { render } from '@ember/test-helpers';
@@ -5,2 +6,13 @@

+const MOCK_URL = new URL(
+  '/foo/bar?baz=true#some-section',
+  window.location.origin
+);
+
+class MockRouterService extends Service {
+  get currentURL() {
+    return '/foo/bar?baz=true#some-section';
+  }
+}
+
 module('Integration | Component | share-button', function (hooks) {
@@ -8,18 +20,20 @@

-  test('it renders', async function (assert) {
-    // Set any properties with this.set('myProperty', 'value');
-    // Handle any actions with this.set('myAction', function(val) { ... });
-
-    await render(hbs`<ShareButton />`);
-
-    assert.dom(this.element).hasText('');
+  hooks.beforeEach(function () {
+    this.owner.register('service:router', MockRouterService);
+  });

-    // Template block usage:
-    await render(hbs`
-      <ShareButton>
-        template block text
-      </ShareButton>
-    `);
+  test('basic usage', async function (assert) {
+    await render(hbs`<ShareButton>Tweet this!</ShareButton>`);

-    assert.dom(this.element).hasText('template block text');
+    assert
+      .dom('a')
+      .hasAttribute('target', '_blank')
+      .hasAttribute('rel', 'external nofollow noopener noreferrer')
+      .hasAttribute(
+        'href',
+        `https://twitter.com/intent/tweet?url=${encodeURIComponent(MOCK_URL.href)}`
+      )
+      .hasClass('share')
+      .hasClass('button')
+      .containsText('Tweet this!');
   });
```

In this component test, we *[registered](../../../applications/dependency-injection/#toc_factory-registrations)* our own router service with Ember in the `beforeEach` hook. When our component is rendered and requests the router service to be injected, it will get an instance of our `MockRouterService` instead of the built-in router service.

This is a pretty common testing technique called *mocking* or *stubbing*. Our `MockRouterService` implements the same interface as the built-in router service – the part that we care about anyway; which is that it has a `currentURL` property that reports the current "logical" URL. This allows us to fix the URL at a pre-determined value, making it possible to easily test our component without having to navigate to a different page. As far as our component can tell, we are currently on the page `/foo/bar/baz?some=page#anchor`, because that's the result it would get when querying the router service.

By using service injections and mocks, Ember allows us to build *[loosely coupled][TODO: link to loosely coupled]* components that can each be tested in isolation, while acceptance tests provide end-to-end coverage that ensures that these components do indeed work well together.

```run:command hidden=true cwd=super-rentals
ember test --path dist
git add tests/integration/components/share-button-test.js
```

While we are here, let's add some more tests for the various functionalities of the `<ShareButton>` component:

```run:file:patch lang=js cwd=super-rentals filename=tests/integration/components/share-button-test.js
@@ -3,3 +3,3 @@
 import Service from '@ember/service';
-import { render } from '@ember/test-helpers';
+import { find, render } from '@ember/test-helpers';
 import { hbs } from 'ember-cli-htmlbars';
@@ -22,2 +22,8 @@
     this.owner.register('service:router', MockRouterService);
+
+    this.tweetParam = (param) => {
+      let link = find('a');
+      let url = new URL(link.href);
+      return url.searchParams.get(param);
+    };
   });
@@ -31,6 +37,3 @@
       .hasAttribute('rel', 'external nofollow noopener noreferrer')
-      .hasAttribute(
-        'href',
-        `https://twitter.com/intent/tweet?url=${encodeURIComponent(MOCK_URL.href)}`
-      )
+      .hasAttribute('href', /^https:\/\/twitter\.com\/intent\/tweet/)
       .hasClass('share')
@@ -38,2 +41,50 @@
       .containsText('Tweet this!');
+
+    assert.strictEqual(this.tweetParam('url'), MOCK_URL.href);
+  });
+
+  test('it supports passing @text', async function (assert) {
+    await render(
+      hbs`<ShareButton @text="Hello Twitter!">Tweet this!</ShareButton>`
+    );
+
+    assert.strictEqual(this.tweetParam('text'), 'Hello Twitter!');
+  });
+
+  test('it supports passing @hashtags', async function (assert) {
+    await render(
+      hbs`<ShareButton @hashtags="foo,bar,baz">Tweet this!</ShareButton>`
+    );
+
+    assert.strictEqual(this.tweetParam('hashtags'), 'foo,bar,baz');
+  });
+
+  test('it supports passing @via', async function (assert) {
+    await render(hbs`<ShareButton @via="emberjs">Tweet this!</ShareButton>`);
+    assert.strictEqual(this.tweetParam('via'), 'emberjs');
+  });
+
+  test('it supports adding extra classes', async function (assert) {
+    await render(
+      hbs`<ShareButton class="extra things">Tweet this!</ShareButton>`
+    );
+
+    assert
+      .dom('a')
+      .hasClass('share')
+      .hasClass('button')
+      .hasClass('extra')
+      .hasClass('things');
+  });
+
+  test('the target, rel and href attributes cannot be overridden', async function (assert) {
+    await render(
+      hbs`<ShareButton target="_self" rel="" href="/">Not a Tweet!</ShareButton>`
+    );
+
+    assert
+      .dom('a')
+      .hasAttribute('target', '_blank')
+      .hasAttribute('rel', 'external nofollow noopener noreferrer')
+      .hasAttribute('href', /^https:\/\/twitter\.com\/intent\/tweet/);
   });
```

The main goal here is to test the key functionalities of the component individually. That way, if any of these features regresses in the future, these tests can help pinpoint the source of the problem for us. Because a lot of these tests require parsing the URL and accessing its query params, we setup our own `this.tweetParam` test helper function in the `beforeEach` hook. This pattern allows us to easily share functionality between tests. We were even able to refactor the previous test using this new helper!

With that, everything should be good to go, and our `<ShareButton>` component should now work everywhere!

```run:command hidden=true cwd=super-rentals
ember test --path dist
git add tests/integration/components/share-button-test.js
```

```run:screenshot width=1024 height=960 retina=true filename=pass-2.png alt="All the tests pass!"
visit http://localhost:4200/tests?nocontainer&nolint&deterministic
wait  #qunit-banner.qunit-pass
```

```run:server:stop
ember server
```

```run:checkpoint cwd=super-rentals
Chapter 10
```
