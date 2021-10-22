---
layout: blog-post
title:  "Automating app store screenshots"
date:   2017-10-02 00:00:00
categories: android java testing
image-base: /assets/images/posts/2017-10-02-automating-app-store-screenshots
---

If there is one part of developing an Android app that I find tedious it has to be updating the screenshots for the store listing after a new release. The drudgery of deciding how I want the app to look for each screenshot, manually preparing it with data and then taking the screenshots, has finally driven me to automate the process.

## Technologies

My Android app is already making use of the following frameworks, so it made sense to reuse them.

- [AndroidJUnitRunner](https://developer.android.com/training/testing/junit-runner.html) - Runs the instrumentation tests.
- [UI Automator](https://developer.android.com/training/testing/ui-automator.html) - In this context it is only used for taking screenshots.
- [Espresso](https://developer.android.com/training/testing/espresso/index.html) - Controls the app during UI tests; reliably clicking buttons and typing text etc.
- [GreenCoffee](https://github.com/mauriciotogneri/green-coffee) - Runs acceptance tests written in Gherkin using the AndroidJUnitRunner.

## Preparing the app for screenshots

The following is an example [Gherkin](https://en.wikipedia.org/wiki/Cucumber_(software)#Gherkin_language) (Given-When-Then) scenario that I use for taking the screenshots. The end result is the app configured, rotated and fed with dummy data via a mocked sensor, ready for the last step to take a screenshot…

```cucumber
Given I have turned on 'Detailed Readings' in the Preferences
And the device has been rotated
When the sensor provides random values with 150ms intervals aligned to:
  | Values | X | Y | Z |
  | 20     | 5 | 3 | 1 |
  | 6      | 8 | 3 | 1 |
  | 50     | 5 | 3 | 1 |
Then a screenshot named 'orientation_1_detailed_spike.png' is saved in the directory 'test-screenshots'
```

![Animation of the app following the scenario's steps]({{ page.image-base }}/animated_scenario.gif)

### Taking the screenshots

The most pertinent step above is the last, which takes the screenshot of the app. This is performed using UI Automator’s convenient [`takeScreenshot`](https://developer.android.com/reference/android/support/test/uiautomator/UiDevice.html#takeScreenshot%28java.io.File%29) method, and is encapsulated as the following Step…

```java
public class ScreenshotSteps extends GreenCoffeeSteps {

    private static final String TAG = "ScreenshotSteps";

    private final UiDevice uiDevice;

    public DeviceSteps(final UiDevice uiDevice) {
        this.uiDevice = uiDevice;
    }

    @Then("^a screenshot named '(.*)' is saved in the directory '(.*)'$")
    public void a_screenshot_named_x_is_saved_in_the_directory_x(final String filename, final String directory) throws Exception {
        final File screenshotDir = new File(
            Environment.getExternalStorageDirectory().getPath(),
            directory
        );

        if(!screenshotDir.exists() && !screenshotDir.mkdirs()) {
            throw new IOException("Failed to create directory: " + screenshotDir);
        }

        final File screenshotFile = new File(screenshotDir, filename);
        Log.i(TAG, String.format("Saving screenshot: %s", screenshotFile));

        if (!uiDevice.takeScreenshot(screenshotFile)) {
            throw new IOException(String.format(
                "Failed to save screenshot: %s", screenshotFile
            ));
        }
    }
}
```

## Transferring the screenshots

If using a service like AWS Device Farm the [screenshots can be automatically saved](http://docs.aws.amazon.com/devicefarm/latest/developerguide/test-types-android-instrumentation.html#test-types-android-instrumentation-screenshots) and presented after each run. Such a service is great if you want screenshots from any of the 200+ Android devices they have available.

Locally though, when run from an Android Virtual Device, the screenshots can be transferred over with ADB’s `pull` command.

```bash
$ adb pull /storage/sdcard/test-screenshots/ ~/Desktop/
```

## Conclusion
Since automating the taking of screenshots I’ve integrated it into the app’s continuous delivery pipeline which means when a new feature is merged I am presented with screenshots like the following; and given I’m using AWS Device Farm the screenshots are taken on the most popular devices for my app.

![alt text]({{ page.image-base }}/example_screenshots.png "Two screenshots of my app")
