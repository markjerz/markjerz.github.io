---
title: How to scrape Twitter followers
---

Scraping the Twitter followers of an account can be really useful for market research, marketing or just to follow a bunch of people that someone else does.

In this article, we use JavaScript and a browser to easily scrape the followers of a particular account.

Step 1. Open Twitter, go to the Followers tab of the account you want to scrape.

Step 2. Press F12 to open the dev tools.

Step 3. Copy/paste the following script in to the console, press Enter, wait for it to scroll through and to say "Done" in the console:

```
var all = [];
var handles = {};
var iters = 0;
var currentScroll = document.documentElement.scrollTop;
  
function process() {
    var followerCells = document.querySelectorAll('[data-testid=primaryColumn] [data-testid=UserCell]'); 
    var some = false;
    Array.from(followerCells).forEach(x => {
      var content = x.children[0].children[1];
      var titleLink = content.querySelector('a[href]');
      var name = titleLink.textContent;
      var handle = titleLink.href.replace('https://twitter.com/', '');
      var desc = x.children.length && x.children[0].children.length > 1 && x.children[0].children[1].children.length > 1 ? x.children[0].children[1].children[1].textContent : '';
      if (!handles.hasOwnProperty(handle)) {
        handles[handle] = true;
        all.push({name, handle, desc});
        some = true;
      }
    });

    currentScroll = document.documentElement.scrollTop;
    window.scrollBy({
        top: 300,
        left: 0,
        behavior: "smooth",
      });
    iters++;
}

document.addEventListener('scrollend', function () {
    if (iters > 1000 || currentScroll == document.documentElement.scrollTop) {
        console.log('Done');
        console.log(JSON.stringify(all));
        return;
    }

    process();
});
process();
```

That's it! You now have a JSON dump of the followers of that account. Copy/paste it in to your program of choice for further processing.

For example, I took that output and imported in to Excel!