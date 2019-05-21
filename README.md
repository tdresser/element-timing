# Element Timing: Explainer

The **Element Timing** API enables monitoring when large or developer-specified image elements or groups of text nodes (Groups of text nodes isn't particularly clear. Can we be more specific?) are displayed on screen.


### Objectives

1.  Inform developers when specific 'hero' (I don't think the word "hero" adds anything here) elements are first displayed on the screen. We support the following image elements: \<img\>, \<image\> inside \<svg\>, and background-images (background images aren't image elements. Maybe "We support the following types of image"?). We also support observing certain groups of text nodes. Web developers understand which images and text on their sites are critical, so after annotating them the browser can provide timing information for the most important elements. Support for images and text enables the majority of web content to be timed via this API.
1.  Enable analytics providers to measure display time of key images or text, without explicit opt in from web developers. In many cases it's not feasible for developers to modify their HTML just to get better performance insights, so it's important to provide basic information even for websites that cannot annotate their elements.


### How do we register elements for observation?

There are two ways an element can be registered for observation. The first way is explicitly via the `elementtiming` HTML attribute. Having this attribute set will signal the browser to expose render timing information for this element (its source image, background image, and/or text). It should be noted that setting the `elementtiming` attribute does not work retroactively: once an element has loaded and is rendered, setting the attribute will have no effect. Thus, it is strongly encouraged to set the attribute before the element is added to the document (in HTML, or if created in JavaScript, before adding it to the document). Having the attribute implies that:

* If the element is an image, there will be an entry for the image.
* If the element is affected by (possibly multiple) background images, there will be one entry for each of those background images.
* If the element is associated to at least one text node and it is implicitly registered for observation (why does it need to be implicitly registered?), there will be one entry for the associated text nodes.

The second way is implicitly: when the element content takes a large portion of the viewport at the time it is first displayed. That is, when an image (which could also be a background-image) occupies a large portion of the viewport, then an entry is dispatched for it. Similarly, when the area occupied by the text nodes associated to an element is large, then an entry is dispatched for that group of text. We register a subset of images and text by default to allow RUM analytics providers to gather information without having to request HTML changes from sites.

TODO: fix implicit registration. (TODO! I haven't looked over this.)

### Image considerations

We define the **image rendering timestamp** as the next paint that occurs after the image has become fully loaded. This is important to distinguish as progressively rendered images may have multiple initial renderings before the image has even been fully received by the user agent.

Allowing third-party origins to measure the time an arbitrary image resource takes to render could expose sensitive information such as whether a user is logged into a website. Therefore, for privacy and security reasons, the <em>image rendering timestamp</em> is only exposed in entries corresponding to resources that pass the [timing allow check](https://w3c.github.io/resource-timing/#dfn-timing-allow-check). However, to enable a more holistic picture, the rest of the information is exposed for arbitrary images.

### Text considerations

We say that a text node **belongs to** its [containing block](https://www.w3.org/TR/CSS2/visudet.html#containing-block-details). This means that an element could have 0 or many associated text nodes with it.

We say that an element is **text-painted** if the following two conditions are satisfied:

* At least one text node <em>belongs to</em> the element.
* All of the text nodes which belong to the element have been painted at least once. (Should be "At least one of the text nodes")

Thus, the **text rendering timestamp** of an element is the time when it becomes <em>text-painted</em>.

Given a group of text nodes, we define the **text rect** as the geometric union of the text node display rectangles within the viewport. (The geometric union wouldn't give us a single rect, would it? I think we need the bounding box of the text nodes?)

### What information is exposed?

A `PerformanceElementTiming` entry has the following attributes:
* `name`: for images, the initial URL for the resource request. For text: the first characters of the associated text.
* `entryType`: it will always be the string "element".
* `startTime`: for images, the <em>image rendering timestamp</em>, or 0 when the resource does not pass the [timing allow check](https://w3c.github.io/resource-timing/#dfn-timing-allow-check). For text, the <em>text rendering timestamp</em>.
* `duration`: it will always be set to 0.
* `intersectionRect`: for images, the display rectangle of the image within the viewport. For text, the <em>text rect</em> of the associated text.
* `responseEnd`: for images, the timestamp of when the last byte of the resource response was received, same as ResourceTiming's [responseEnd](https://w3c.github.io/resource-timing/#dom-performanceresourcetiming-responseend). For text, 0.
* `identifier`: the value of the `elementtiming` attribute of the element.
* `naturalWidth`: the [intrinsic](https://drafts.csswg.org/css2/conform.html#intrinsic) width of the image. It matches with the corresponding DOM [attribute](https://html.spec.whatwg.org/multipage/embedded-content.html#dom-img-naturalwidth) for img. 0 for text.
* `naturalHeight`: the [intrinsic](https://drafts.csswg.org/css2/conform.html#intrinsic) height of the image. It matches with the corresponding DOM [attribute](https://html.spec.whatwg.org/multipage/embedded-content.html#dom-img-naturalheight) for img. 0 for text.
* `id`: the element's ID.
* `element`: points to the element. This will be "null" if the element is [disconnected](https://dom.spec.whatwg.org/#connected).

Note: for background images, the element is the one being affected by the background image style.

Sample code:

```
<img src="my_image.jpg" elementtiming="foobar">

const observer = new PerformanceObserver((list) => {
  let perfEntries = list.getEntries().forEach(function(entry) {
      // Send the information to analytics, or in this case just log it to console.
      // |entry.startTime| contains the timestamp of when the image is displayed.
      if (entry.identifier === 'foobar')
        console.log("My image took " + entry.startTime + " to render!");
   });
});
observer.observe({entryTypes: ['element']});
```

### Questions

#### What about Shadow DOM?

TODO

#### What about invisible or occluded elements?

The entry creation might be affected by _visibility_: for instance, elements are not exposed if the style visibility is set to none, or the opacity is 0. However, occlusion will not affect entry creation: an entry is seen if the element is there, but hidden by a full-screen pop-up on the page.

#### What are some differences between text and image observation?

* Whereas for images it is OK to care only about those which are associated to a resource, for text that is definitely not the case. An image not associated to a resource will need to be constructed manually and it is uncommon for such an image to be part of the key content of a website. (Might be worth mentioning data URIs, as this sounds like it might exclude them) On the other hand, text is relevant regardless of whether it is associated to a webfont or not.

* Image rendering steps are different from text rendering steps. For images, initial paints may not include all of the image content and may be low quality. It is only once we have fully loaded and decoded the image that we can be certain that the content being displayed is meaningful. In contrast, any text painted is meaningful because there is no such thing as painting partial text (Might want to expand on the case where a block level element contains text nodes which both have and have not painted). Text may still have various rendering steps, but that just happens when webfonts take a while to load and the browser uses a fallback font to render text so that the user can access the content earlier.

* Text is actually simpler when it comes to security and privacy considerations. Cross-origin images could reveal a lot of information about users, but text and webfonts embedded in a website do not. Therefore, whereas for images we had to consider resources that passed the timing allow check versus resources that did not, for text this distinction is not needed.

