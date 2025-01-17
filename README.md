# What is this?

It's a web extension for Firefox or Chrome that allows to add comments to any URL.

![](docs/screenshot.png)

More precisely, it attaches comments to SHA1 of the URL, to avoid leaking PII:

```
   +----------------------+
   | The current tab URL: |
   | http://example.com/  |
   +----------------------+
              |
              | 1. URL
              |
              v                            +-------------------------------------+
  +--------------------+                   | <iframe> with URL:                  |
  | This web extension | ----------------->| https://comntr.github.io/#<tab-url> |
  +--------------------+                   +-------------------------------------+
              |                                            |
              | -> 2. GET /<sha1>/size                     |
              | <- 12 comments                             | -> 3. GET /<sha1>
              |                                            | <- JSON with comments
              v                                            | -> 4. POST /<sha1>
   +-----------------+                                     |    Hello World!
   | Database Server | <-----------------------------------+
   +-----------------+
```

1. The user opens a URL and the extension gets this URL via the web extension API.
2. The extension computes SHA1 of the URL and sends a `GET /<sha1>` request to the database to get the number of comments. Then this number is displayed in the extension icon's badge (just like uBlock Origin shows the number of blocked elements).
3. The user clicks on the extension icon to see the comments. The extension sends a `GET /<sha1>` request and the database returns all the comments attached to that SHA1.
4. The user adds a comment and the extension  sends a `POST /<sha1>` with the comment text.

The database server doesn't see the original URLs and uses their SHA1 instead:

```
/var/lib/comntr
  /9c1...d14        # SHA1 of "http://example.com/"
    /261...776      # SHA1 of the comment file
      Hello World!
    /829...172
      Howdy
```

The advantage of the `<iframe>` approach is that the comments can be viewed in a separate tab:

- https://comntr.github.io/#http://example.com/

The extension merely renders this page in the popup.

Involved components:
- This web extension.
- The database server: https://github.com/comntr/http-server
- Page that renders the comments: https://github.com/comntr/comntr.github.io

# Installation

For Firefox dekstop and Firefox Android use https://addons.mozilla.org/en-US/firefox/addon/comntr/. It probably works in Firefox iOS too, but I haven't checked.

The extension works in Chrome too, but it's not published to the Chrome Web Store. Why? Publishing any extension there now requires (1) a phone number, to create a gmail account and (2) a credit card, supposedly to deter spammers. However the extension can be installed manually. First, copy the files to some folder:

```
mkdir -p ~/webext/comntr
cd ~/webext/comntr
git clone https://github.com/comntr/webext .
```

Navigate to `chrome://extensions` in Chrome and find the "Developer mode" toggle in the top right corner. Use "Load unpacked" to select the `~/webext/comntr` folder. The extension should appear on the page and its icon should appear in the top right nav bar. It may be saying that there were some errors, but those are because Chrome didn't like the Firefox-specific field in `manifest.json`:

```
  "browser_specific_settings": {
    "gecko": {
      "id": "webext@comntr.io"
    }
  }
```

This error can be ignored.

# Privacy Policy

This extension tries to not collect any PII. However some PII is still leaked:

- The data server sees your IP address. Although this IP address likely changes every time you connect to your ISP, it can still give an idea where approximately you live. The IP address isn't stored anywhere, as you can see in the data server sources.
- The extension sees the URLs you visit. The URLs aren't sent to the data server. Instead, the extension computes SHA1 of the URL and sends the hash. The data server can theoretically precompute hashes for millions of often used URLs and know if you visit any of those URLs.
- Every comment you send is signed with a ed25519 key. The keys are stored in the browser, in indexedDb, which is flushed to file system from time to time. Since every your comment has the same public key, it's possible to find all the comments you've sent. However you can delete the keys and the extension will generate new ones once you send a comment.
- The extension caches most recently seen comments in indexedDb. Thus any comments that you send, end up not only in the data server's file system, but also in local caches of other users who see your comments.

# Roadmap

A few problems need to be solved before this idea can get any meaningful adoption:

- A widget that website admins can add to their sites. Obviously, they'll want control over what people write there. Comments can be physically stored on the same data server or on their own servers.
- Comments data needs to be open and federated. It should be possible to start your own data server that would join the network. I'm definitely not trying to grab control over all comments in the world. That wouldn't really work anyway.
- Subnetworks with different rules. One big space for everyone won't work because scientists won't be able to coexist with trolls and spammers. Subnetworks may have rules and moderators. In practice, this will look like a few data servers that anyone can subscribe too. There will be a default server that will likely be moderated, but if someone wants to see comments from `data.sci.org`, they would switch to this server in their config.
- A way to make users spend their time (not their CPU time!) to post comments. Without this it'll be hard to stop spammers and trolls. A simple solution might work: the auth server generates a short random number, but returns it as an SVG picture where the digits are drawn with circles or squares. It's easy for the server to generate such images and verify the answers and it's easy for humans to read this, but spammers would have to set up an ML image recognition service, which is way beyond the abilities of most spammers. To deter trolls, the SVG picture can present a basic question like "23+47". The point is to make trolls pause and think and I'd argue that those who can answer this question quickly, aren't trolls. We can raise the bar higher for a math community and present questions like "log(32)/log(2)" - a no brainer for anyone familiar with entry level math, but a hard problem for random people that want to post meaningless comments where they really shouldn't.

# Credits

The ed25519 wasm library was taken from [nazar-pc/supercom.wasm](https://github.com/nazar-pc/supercop.wasm).

The icon was made by Freepik from [www.flaticon.com](https://www.flaticon.com/free-icon/world_523491).
