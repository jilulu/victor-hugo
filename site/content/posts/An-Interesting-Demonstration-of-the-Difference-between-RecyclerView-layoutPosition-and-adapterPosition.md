---
title: >-
  An Interesting Demonstration of the Difference between RecyclerView
  layoutPosition and adapterPosition
date: 2018-07-25 14:31:32
tags: [android, dev]
---

No. The title of this article is quite misleading, but I could not thinks of a better one that remains concise. A view's position in a RecyclerView's layout is definitely different from its position in the adapter attached to it. But we're addressing another subtile topic: the difference between [RecyclerView.ViewHolder](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.ViewHolder)'s [`getLayoutPosition`](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.ViewHolder#getlayoutposition) method and its [`getAdapterPosition`](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.ViewHolder.html#getAdapterPosition()) method. 

## Motivations
Quoting from the Android documentation of `RecyclerView.ViewHolder#getAdapterPosition()` method
> Note that this might be different than the getLayoutPosition() if there are pending adapter updates but a new layout pass has not happened yet.

"Fine, now it's clear to me that this method is different from RecyclerView's `getLayoutPosition()` method. ", that's what occurred to me the first time I read that doc. That being thought, I would still ponder about which between the two to choose from, when required to find a RecyclerView child view's position given a reference to that view. And my solution has eventually become "using neither of the two", because there's a third solution, namely to rely on `LayoutManager`'s [`getPosition`](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.LayoutManager#getposition) method. It's worth noting that no magic is going on here, as its implementation is simply redirecting to `getLayoutPosition()`:  
```java
public int getPosition(View view) {
    return ((RecyclerView.LayoutParams) view.getLayoutParams()).getViewLayoutPosition();
}
```
I thought it was no big deal until the following happened. 

## The messed-up margins
The symptom, in one line, is that RecyclerView item decorations are messed up after child items got swapped. Here's what it looks like: 

![Actual behaviour](/2018/recycler-view-messed.gif)

It's clear that everything else works fine except for the item decoration. After some patient observation one may conclude that the spacing to the left of first item and that of second item are mistakenly swapped. Now inspecting the lines of code responsible for figuring out the offsets of an RecyclerView child item in my ItemDecoration class and after some digging around, I'm about to narrow down the cause of said bug to this one line of code
```kotlin
val position: Int? = parent?.layoutManager?.getPosition(view)
```
The cause of the problem is that layoutManager returns incorrect positions. Or more precisely, it always returns correct positions, but invoked at the wrong time. It turns out that when RecyclerView queries our ItemDecoration implementation for the appropriate offsets, a layout to reflect that change has **not** happened yet. Referring to the docs, `getViewLayoutPosition` is defined so that it
> Returns the adapter position that the view this LayoutParams is attached to corresponds to as of latest layout calculation.

Not actually what we desire right? That's historical data! The fix to this bug, as it has revealed itself, is to change that invocation to `getViewAdapterPosition()`
```
val position: Int? = (view?.layoutParams as? RecyclerView.LayoutParams)?.viewAdapterPosition
```

![Fixed](/2018/recycler-view-fixed.gif)

... and case closed. 