---
layout: post
title: AppCompat v23.2 — Age of the vectors
date: '2016-02-25'
cover_image: '/content/images/2016/rulers.jpg'
---
As you may have seen on the Support Lib 23.2.0 blog post, we now have compatible vector drawable implementations in the support libraries: VectorDrawableCompat and Animated VectorDrawableCompat.

Those are implemented as standalone pieces of functionality. As we know that developers want to use them from resources, we've added support for vector drawables directly into AppCompat.

There are various reasons for this integration, including:

* Allows developers to easily use `<vector>` drawables on all devices running Android 2.1 and above.
* Allows AppCompat to use vector drawables itself. This itself has shaved off around 70KB from AppCompat's AAR (~9%). That doesn't sound like a lot, but the saving is compounded for all apps which use AppCompat, on all devices. The savings quickly add up here for both storage and transfer.

## First things first

VectorDrawableCompat relies on some functionality in aapt which tells it to keep around any recently added attribute IDs which `<vector>` uses, so that they can be referenced pre-v21. This is turned on via an aapt flag (mentioned below).

Without this flag enabled, you will see the following error (or similar) when running your app on devices running KitKat or lower:

    Caused by: android.content.res.Resources$NotFoundException:
        File res/drawable-v19/abc_ic_ab_back_material.xml from drawable resource ID ...
    at android.content.res.Resources.loadDrawable(Resources.java:2097)
    at android.content.res.Resources.getDrawable(Resources.java:700)
    ...

### Enabling the flag

I'm guessing that most of you will be using Gradle so let's quickly walk through that.

If you're using v2.0 or above of the Gradle plugin, we have a handy shortcut:

{% highlight groovy %}
android {
    defaultConfig { 
        vectorDrawables.useSupportLibrary = true  
    }
}
{% endhighlight %}

If you have not updated yet, and are using v1.5.0 or below of the Gradle plugin, you need to add the following to your app's build.gradle:

{% highlight groovy %}
android {
    defaultConfig {
    // Stops the Gradle plugin's automatic rasterization of vectors
        generatedDensities = []  
    }
        
    // Flag to tell aapt to keep the attribute ids around
    aaptOptions {
        additionalParameters "--no-version-vectors"  
    }
}
{% endhighlight %}

## How do I use my own Vector assets in my app?

There are a few things to note before we start. When using AppCompat, VectorDrawableCompat is only used on API 20 and below. This means that you'll be using the framework's VectorDrawable class when running on API 21 and above. This is slightly different to the create() APIs on VectorDrawableCompat which still uses the framework implementation, but via a proxy object.

Right, so you want to use a vector asset in your app and your minSdkVersion is < 21. Great! First check the asset works on a API 21+ device, just as a sanity check.

### AppCompatImageView

So you probably know that AppCompat 'injects' its own widgets in place of many framework widgets. This allows it to perform tinting and other backporty things. We've added support for VectorDrawableCompat here too with the new `app:srcCompat` attribute. You can safely use srcCompat for non-vector assets too.

Here's an example vector asset which we're actually using in AppCompat:

**res/drawable/ic_search.xml**

{% highlight xml %}
<vector xmlns:android="..."
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24.0"
    android:viewportHeight="24.0"
    android:tint="?attr/colorControlNormal">
    
    <path android:pathData="..."
        android:fillColor="@android:color/white" />
            
</vector>
{% endhighlight %}

Using this drawable, an example ImageView declaration would be:

{% highlight xml %}
<ImageView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:srcCompat="@drawable/ic_search" />
{% endhighlight %}

You can also set it at run-time:

{% highlight java %}
ImageView iv = (ImageView) findViewById(...);
iv.setImageResource(R.drawable.ic_search);
{% endhighlight %}

The same attribute and calls work for ImageButton too.

## What about Animated Vectors?

So far we've only talked about 'static' vector drawables. So let's talk about animated vectors. They work too in much the same way, but they're only available on API v11+. If you try to load an `<animated-vector>` on devices running API 10 or below then the system will return `null` and nothing will be displayed.

There are also some limitations to what kind of things animated vectors can do when running on platforms < API 21. The following are the things which do not work currently on those platforms:

* Path Morphing (PathType evaluator). This is used for morphing one path into another path.
* Path Interpolation. This is used to defined a flexible interpolator (represented as a path) instead of the system defined ones like LinearInterpolator.
* Move along path. This is rarely used. The geometry object can move around, along an arbitrary path.

In summary, the Animator declaration has to be valid and functional based on the platform your app will be running on.

---

_Cover photo: [rulers](https://flic.kr/p/mKQ2JR) by Dean Hochman_

[1]: https://developer.android.com/reference/android/content/res/TypedArray.html
[2]: https://android.googlesource.com/platform/frameworks/base/+/refs/heads/kitkat-release/graphics/java/android/graphics/drawable/InsetDrawable.java#87
