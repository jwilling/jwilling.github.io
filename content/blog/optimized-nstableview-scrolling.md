---
title: "Optimized Scrolling in NSTableView"
date: 2013-01-11
aliases:
  - /optimized-scrolling-in-nstableview

---

First introduced in OS X Lion, view-based table views are a significant advancement over previous cell-based table views. Not only are views easier to work with and customize, but they're also trivial to animate. The API is fantastic. So what's the catch?

Scrolling. A standard, unoptimized view-based table view with a somewhat complex layout will have an extremely tough time achieving buttery smooth scrolling at 60 fps. The good news is that we can fix this. Ready? Let's get started.

### It's all about the layers

Views without layers require a redraw after every frame change. Layer-backed views, on the other hand, are able to cache the result of drawing into an bitmap cached in the GPU, which can then animate and transform that bitmap with great ease. `NSView` does not enable layer-backing by default.

One class that is critical in the process of scrolling is `NSClipView`. A clip view is used to contain the document view of a `NSScrollView`, clip the document view to its frame, and update the `NSScrollView` when the document view's size or position changes<sup>[1](https://developer.apple.com/library/mac/#documentation/Cocoa/Reference/ApplicationKit/Classes/NSClipView_Class/Reference/Reference.html)</sup>.

---

**Update 09/22/15**: The information below was written for OS X 10.7. Scroll views as of 10.8+ seem to perform well without needing to customize the layer class of the clip view. I would recommend against swapping the clip view. Rather, the containing scroll view should have `wantsLayer` set to `YES`.

---

If your window's content view, or any superview of the clip view contains a layer, it will implicitly get a layer generated for itself as well. By default it will use a `_NSViewBackingLayer` for the backing layer. Unfortunately, this type of layer isn't quite ideal for a clip view. A much more ideal subclass of `CALayer` is `CAScrollLayer`. A [`CAScrollLayer`](http://developer.apple.com/library/mac/#documentation/GraphicsImaging/Reference/CAScrollLayer_class/Introduction/Introduction.html), as its name implies, is a subclass of `CALayer` that is optimized for scrolling, and displaying a portion of a layer. Wouldn't it be great if we could use this for our `NSClipView`? Well, we can.

Telling a subclass of `NSClipView` to use a `CAScrollLayer` is an easy matter of **overriding the designated initializer, setting the layer to a `CAScrollLayer`, and telling the clip view to use a layer**. Since we create and set the layer before telling the clip view it needs a layer, this is a layer-hosting view.

```objectivec
- (id)initWithFrame:(NSRect)frame {
	self = [super initWithFrame:frame];
	if (!self) return nil;

	self.layer = [CAScrollLayer layer];
	self.wantsLayer = YES;
	self.layerContentsRedrawPolicy = NSViewLayerContentsRedrawNever;

	return self;
}
```

We're not quite there yet. In order to use this fancy clip view, we're going to need to **replace the existing clip view with our custom clip view at runtime**. There are multiple ways to do this. I chose to subclass `NSScrollView` and swap the clip view out during initialization.

```objectivec
- (id)initWithFrame:(NSRect)frameRect {
	self = [super initWithFrame:frameRect];
	if (self == nil) return nil;

	[self swapClipView];

	return self;
}

- (void)awakeFromNib {
	[super awakeFromNib];

	if (![self.contentView isKindOfClass:CustomClipView.class] ) {
		[self swapClipView];
	}
}

- (void)swapClipView {
	self.wantsLayer = YES;
	id documentView = self.documentView;
	CustomClipView *clipView = [[CustomClipView alloc] initWithFrame:self.contentView.frame];
	self.contentView = clipView;
	self.documentView = documentView;
}
```

The layer-backed clip view already exists in Github's [Rebel](https://github.com/github/rebel), and I've added the custom scroll view to Rebel as well. I highly recommend using it.

Ideally, scrolling should be quite a bit better now. But we're not done yet.


### Auto Layout

The idea behind Auto Layout is fantastic. However, with view-based table cells that require somewhat complex layout, auto layout has the potential to cause quite a bit of lag. The only way to remove all auto layout from the table view is to disable auto layout *for your entire window.* Yes, it's not ideal. But once auto layout is triggered for any view in your window, the layout system is enabled for every other view in the window, including your table view.

Now let me reiterate: this is only necessary for cells that require a significant amount of layout customization. If you just have an image and a couple of labels, this might not be necessary. **Profiling is the only way to determine for sure if auto layout is your problem.**


### Drawing

If you want a custom cell appearance, the recommended way to approach this is to subclass `NSTableRowView`, override one of the drawing methods (such as `-drawBackgroundInRect:`), and draw the custom appearance.

When a table cell view (or a table row view) is dequeued, it is removed from its current superview, and re-added once it has been appropriately reset and enqueued. This means that while scrolling your table view, `-drawBackgroundInRect:` is being called repeatedly every time the row view is dequeued.

A better approach would be to **cache your background as a stretchable image**. This image can then be simply set to the layer's contents on your row view or cell view. By reusing the same image, you're using the same bitmap in memory for all the rows, and you're avoiding expensive drawing of your background.

If you're using 10.8, the most optimized way to do this is to override `-wantsLayer` and return `YES`, which allows AppKit in turn to call `-updateLayer`. This is the recommended time to set the layer's contents.


### What else?

Listed above are just a few techniques I've discovered that help make scrolling more smooth. In a future post I'll detail how scrolling using the arrow keys can be recreated to provide a smoother animation.
