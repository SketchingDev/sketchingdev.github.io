---
layout: blog-post
title:  "Testing activity redirection in Android apps"
date:   2017-07-11 00:00:00
categories: android testing java
---

I am currently in the process of adding a dialog to my Android app which asks if the user would rate it via the
Google Play Store. However, writing acceptance tests around this logic has been problematic, as starting the Play Store
in Android's instrumentation testing framework causes my app to lose focus and the tests to hang as a result.

## Quick solution: Mocking

The quick solution, which is better suited to unit/integration tests, is to encapsulate the code that starts the
activity and mock it to prevent the Play Store from opening. The problem with this though (as simple as it is) is
that for acceptance testing we want to keep the code as close to what will run in production as possible, whereas
mocking hides logic.

```java
public class PlayStoreLauncher implements StoreLauncher {
    private final Intent intent = new Intent(/* ... */);

    @Override
    public void open(Context c) {
        c.startActivity(intent);
    }
}

public class MainActivity extends AppCompatActivity {
    private StoreLauncher launcher;

    public void setStoreLauncher(StoreLauncher l) {
        launcher = l;
    }

    public void openStore() {
        launcher.open(this)
    }

    //...
}
```

## Better solution: ActivityMonitor

A better solution which I ultimately used, is to block the Play Store activity from opening by using the
instrumentation's ActivityMonitor class.
This way all the logic remains untouched, but the ActivityMonitor, once configured, will block the intent from opening
a new activity. You’re then able to use the same class to verify that the activity would have been opened.

```java
public class RatingRequestTests {

    private static final boolean BLOCK_ACTIVITY_STARTING = true;

    @Rule
    public final ActivityTestRule<MainActivity> activityRule = new ActivityTestRule<>(MainActivity.class, false, true);

    private Instrumentation instrumentation;

    private Instrumentation.ActivityMonitor activityMonitor;

    @Before
    public void setup() {
      instrumentation = InstrumentationRegistry.getInstrumentation();
      activityMonitor = new Instrumentation.ActivityMonitor(
          createPlayStoreIntentFilter(),
          null,
          BLOCK_ACTIVITY_STARTING);
    }

    @Test
    public void playstore_opened_when_rating_request_accepted() {
        instrumentation.addMonitor(activityMonitor);

        new MainScreenObjectModel().acceptRatingRequest();
        assertEquals(activityMonitor.getHits(), 1);

        instrumentation.removeMonitor(activityMonitor);
    }

    private static IntentFilter createPlayStoreIntentFilter() {
        // ...
    }
}
```
