---
layout: post
title: Some quick fixes for the oudated Google Chrome Extension Tutorial
type: post
published: true
comments: true
---

If you're following this [guide](https://developer.chrome.com/extensions/getstarted)
to learn how to write extensions, you'll find that it is seriously
oudated. With some effort though that can be fixed, and that's what I'll
describe how to do here.

### Register with Google API

If you're starting with the overview, you'll notice that after putting
together the extension and loading it in your browser, the extension
will not load any images in its popup window.

The first thing to understand is that the API used in that tutorial is
deprecated (by Google themselves). There's still a way to access the
search API according to this
[forum post](https://stackoverflow.com/questions/4082966/what-are-the-alternatives-now-that-the-google-web-search-api-has-been-deprecated).

### Figure out your request credentials
Once you have a key and account on [CSE](https://cse.google.com)
configure the custom search according to the instructions above. Enable
images, set it search the entire web, etc...then using the search box on
the right side of the control panel, search anything, and then click the
images tab to see those results. Do this with the network tab of your
browser's inspector window open. From there, copy the network request
URL request with all of its keys and parameters.

![alt text]({{ site.url }}/assets/img/chrome-inspector.png "inspector after clicking
the image tab")

### Edit your popup.js
Once you know you've got all your parameters, edit your `popup.js` file
to do a couple of things.
* Fix your `searchUrl` variable by adding the query url you got from
the inspector
* Remove the `!response.responseData` condition from the if statement on
line 71
* Remove any instance of `.responseData` in the code. The search results
nowadays are stored in `response.results`

Your code should look something like this:
```js
// Copyright (c) 2014 The Chromium Authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

/**
 * Get the current URL.
 *
 * @param {function(string)} callback - called when the URL of the current tab
 *   is found.
 */
function getCurrentTabUrl(callback) {
    // Query filter to be passed to chrome.tabs.query - see
    // https://developer.chrome.com/extensions/tabs#method-query
    var queryInfo = {
        active: true,
        currentWindow: true
    };

    chrome.tabs.query(queryInfo, function(tabs) {
        // chrome.tabs.query invokes the callback with a list of tabs that match the
        // query. When the popup is opened, there is certainly a window and at least
        // one tab, so we can safely assume that |tabs| is a non-empty array.
        // A window can only have one active tab at a time, so the array consists of
        // exactly one tab.
        var tab = tabs[0];

        // A tab is a plain object that provides information about the tab.
        // See https://developer.chrome.com/extensions/tabs#type-Tab
        var url = tab.url;

        // tab.url is only available if the "activeTab" permission is declared.
        // If you want to see the URL of other tabs (e.g. after removing active:true
        // from |queryInfo|), then the "tabs" permission is required to see their
        // "url" properties.
        console.assert(typeof url == 'string', 'tab.url should be a string');

        callback(url);
    });

    // Most methods of the Chrome extension APIs are asynchronous. This means that
    // you CANNOT do something like this:
    //
    // var url;
    // chrome.tabs.query(queryInfo, function(tabs) {
    //   url = tabs[0].url;
    // });
    // alert(url); // Shows "undefined", because chrome.tabs.query is async.
}

/**
 * @param {string} searchTerm - Search term for Google Image search.
 * @param {function(string,number,number)} callback - Called when an image has
 *   been found. The callback gets the URL, width and height of the image.
 * @param {function(string)} errorCallback - Called when the image is not found.
 *   The callback gets a string that describes the failure reason.
 */
function getImageUrl(searchTerm, callback, errorCallback) {
    // Google image search - 100 searches per day.
    // https://developers.google.com/image-search/

    var searchUrl = 'https://www.googleapis.com/customsearch/v1element?key=KEYGOESHERE&num=20&hl=en&sig=SIGGOESHERE&searchtype=image&cx=CXGOESHERE&cse_tok=TOKENGOESHERE' +
        '&q=' + encodeURIComponent(searchTerm);
    var x = new XMLHttpRequest();
    x.open('GET', searchUrl);
    // The Google image search API responds with JSON, so let Chrome parse it.
    x.responseType = 'json';
    x.onload = function() {
        // Parse and process the response from Google Image Search.
        var response = x.response;
        if (!response || !response.results || response.results.length === 0) {
            errorCallback('No response from Google Image search!');
            return;
        }
        var firstResult = response.results[0];
        // Take the thumbnail instead of the full image to get an approximately
        // consistent image size.
        var imageUrl = firstResult.tbUrl;
        var width = parseInt(firstResult.tbWidth);
        var height = parseInt(firstResult.tbHeight);
        console.assert(
            typeof imageUrl == 'string' && !isNaN(width) && !isNaN(height),
            'Unexpected respose from the Google Image Search API!');
        callback(imageUrl, width, height);
    };
    x.onerror = function() {
        errorCallback('Network error.');
    };
    x.send();
    console.log('request sent');
}

function renderStatus(statusText) {
    document.getElementById('status').textContent = statusText;
}

document.addEventListener('DOMContentLoaded', function() {
    getCurrentTabUrl(function(url) {
        // Put the image URL in Google search.
        renderStatus('Performing Google Image search for ' + url);

        getImageUrl(url, function(imageUrl, width, height) {

            renderStatus('Search term: ' + url + '\n' +
                'Google image search result: ' + imageUrl);
            var imageResult = document.getElementById('image-result');
            // Explicitly set the width/height to minimize the number of reflows. For
            // a single image, this does not matter, but if you're going to embed
            // multiple external images in your page, then the absence of width/height
            // attributes causes the popup to resize multiple times.
            imageResult.width = width;
            imageResult.height = height;
            imageResult.src = imageUrl;
            imageResult.hidden = false;

        }, function(errorMessage) {
            renderStatus('Cannot display image. ' + errorMessage);
        });
    });
});
```

I know, it's probably not even worth fixing it but it was a fun exercise
 and I thought I'd share it anyways.

 ![alt text]({{ site.url }}/assets/img/chrome-extension-works.png)
