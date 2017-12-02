---
layout: post
title:  "How ProGuard protects Android applications from reverse engineering"
date:   2015-11-05 00:00:00
categories: android security
image-base: /assets/images/posts/2015-11-05-how-proguard-protects-android-applications-from-reverse-engineering
---

[ProGuard](http://proguard.sourceforge.net/) is Java class file shrinker, optimizer, obfuscator and preverifier which is baked into [Android Studio's](https://developer.android.com/sdk/index.html) build process. Because it's part of the build process you have to go out your way to see what it actually does to your code and how it would look to someone trying to reverse engineer your application.

Whilst we could just run ProGuard against a compiled Java class I thought it would be reassuring to recreate the process a miscreant might take to dissemble someone's APK, and see what ProGuard has done to protect the code.

Enabling ProGuard in a project is done by explicitly setting the `minifyEnabled` property in your Gradle file to `true`.

```groovy
android {
    ...
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```

## Building the APK

There are [many steps to the build process](https://github.com/dogriffiths/HeadFirstAndroid/wiki/How-Android-Apps-are-Built-and-Run) however there are only four that we're interested in. In order to dissemble the resulting APK we'll have to reverse each of these processes.

![Java Decompiler screenshot]({{ page.image-base }}/build-process.png)

1. Compile the Java code
2. Run ProGuard against the compiled Java classes
3. Compile Java classes into Dalvik byte-code
4. Package resulting .dex files into an APK

We can see the build process in action by cloning the [demonstration project](https://github.com/SketchingDev/ProGuard-Demonstration) that I created and then calling the `assembleRelease` gradle task, which will build the release APK without signing it.

```bash
$ git clone https://github.com/SketchingDev/ProGuard-Demonstration.git
$ cd ProGuard-Demonstration/

$ ./gradlew assembleRelease
```

Once the 'assembleRelease' task has finished then the location of the APK is usually './app/build/outputs/apk/app-release-unsigned.apk'.

## Dissembling the APK

Now that we've built the APK we can reverse the build process - with the help of some open-source tools - to see what the Java code looks like after ProGuard has done its job.

<div class="fillwidth">
<img src="{{ page.image-base }}/dissemble-process.png" alt="Java Decompiler screenshot" />
</div>

1. Unzip the APK and extract the dex file
2. Use [dex2jar](https://github.com/pxb1988/dex2jar/releases) to dissemble the dex to a Jar file
3. Use a [JD-GUI](http://jd.benow.ca/) to decompile the Java classes in the Jar.

```bash
$ unzip -p ./app/build/outputs/apk/app-release-unsigned.apk 'classes.dex' > ~/classes.dex

$ d2j-dex2jar ~/classes.dex -o ~/classes.jar
```

After dex2jar produces the '~/classes.jar' file you can open it with [JD-GUI](http://jd.benow.ca/) to decompile the Java code to see what it now looks like post-ProGuard.

![Java Decompiler screenshot]({{ page.image-base }}/java-decompiler.png)

### Before and After

Although this piece of code is quite simple it is easy to see how ProGuard has made it harder to read by using scoped name obfuscation and removing unused methods (in this case the `dummyMethod()`).

#### Before code obfuscation

```java
public class RandomNameGenerator {

	private final Random random = new Random();

	private String[] mFirstNames;
	private String[] mLastNames;

	public RandomNameGenerator(String[] firstNames, String[] lastNames) {
		mFirstNames = firstNames;
		mLastNames = lastNames;
	}

	public String[] getName() {
		int firstNameIndex = random.nextInt(mFirstNames.length);
		int lastNameIndex  = random.nextInt(mLastNames.length);

		return new String[] { mFirstNames[firstNameIndex], mLastNames[lastNameIndex] };
	}

	// This method isn't used, so let's see what ProGuard does with it
	public void dummyMethod() {}
}
```

#### After code obfuscation

```java
public class b
{
  private final Random a = new Random();
  private String[] b;
  private String[] c;

  public b(String[] paramArrayOfString1, String[] paramArrayOfString2)
  {
    this.b = paramArrayOfString1;
    this.c = paramArrayOfString2;
  }

  public String[] a()
  {
    int i = this.a.nextInt(this.b.length);
    int j = this.a.nextInt(this.c.length);
    return new String[] { this.b[i], this.c[j] };
  }
}
```
