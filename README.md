# flutter_service_worker_bug

## Problem description

Some external requests are processed by the service worker in Flutter for web and other requests are not, depending on the length of the application URL in combination with the request URL.

For example, given an application URL: https://www.some-website-example.com that makes the following requests:

- A GET request to https://www.some-api.com/status
- A GET request to https://www.another-api.com/items/12345

The first request will be processed (and cached) by the service worker, while the second one will not.

The expected behavior of the service worker is that it should only handle application resources and must never interfere with external requests.

## Prerequisites

Ensure you have a tool installed that can run an HTTP server to serve the contents of a directory. In this README, Python 3 is used as an example. Other examples can be found at https://gist.github.com/willurd/5720255.

## Getting started

1. Clone this repository.
2. Install dependencies: `flutter pub get`.
3. Compile for web: `flutter build web`
    - Note that `flutter run -d chrome` is not sufficient, since service workers are empty in `run` mode.
4. Serve the output, e.g., using `python3 -m http.server 8080 -d build/web`.
5. Open the following URLs in Chrome:
    - http://this-is-a-very-long-url-thats-sooooooooooooooooooooooooooo-long.localhost:8080/
    - http://short.localhost:8080/
6. Open the network tools.
7. In each tab, press the download (FAB) button.
8. Observe that for the long URL, the request is processed by the service worker (a ⚙ is present), and for the short URL, it is not.

## Cause of this discrepancy

Looking into the service worker code, we find the following code: [link](https://github.com/flutter/flutter/blob/2528fa95eb3f8e107b7e28651a61305d960eef85/packages/flutter_tools/lib/src/web/file_generators/js/flutter_service_worker.js#L87-L95):

```js
  var origin = self.location.origin;
  var key = event.request.url.substring(origin.length + 1);
  // Redirect URLs to the index.html
  if (key.indexOf('?v=') != -1) {
    key = key.split('?v=')[0];
  }
  if (event.request.url == origin || event.request.url.startsWith(origin + '/#') || key == '') {
    key = '/';
  }
```

Depending on the length of the URLs, the `substring` returns the empty string, which later on becomes `/`.

### Suggested fix

Bail out when the request is to an external URL. For example, using:

```js
  var requestURL = new URL(event.request.url);
  if (requestURL.origin !== origin) {
    return;
  }
```