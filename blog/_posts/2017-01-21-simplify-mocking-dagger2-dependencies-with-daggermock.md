---
layout: post
title:  "Simplify mocking Dagger2 dependencies with DaggerMock"
date:   2017-01-21 00:00:00
categories: android testing java
---

Dagger2 is a great improvement over its predecessor, but tasks like mocking dependencies for system tests can still be
quite verbose to setup, which is where the [DaggerMock](https://github.com/fabioCollini/DaggerMock) library comes in
handy!

Before DaggerMock trying to provide mocked objects required (at the very least) a new module that will provide the
mocked dependency and then depending on the approach you take having to build a dependency graph that includes your
mocked module and all other non-mocked modules. The test below taken from a
[demonstration project](https://github.com/SketchingDev/DaggerMock-Demonstration) I put together  shows just some of the
extra work just to inject a mocked `ButtonTextService`.

```java
public class TestButtonInUi extends UiTest {

    @Singleton
    @Component(modules = {AppModule.class, MockedButtonModule.class})
    interface TestButtonInUiAppComponent extends AppComponent {
        void inject(TestButtonInUi activity);
    }

    @Rule
    public final ActivityTestRule<MainActivity> activityRule = new ActivityTestRule<>(MainActivity.class, false, false);

    @Inject
    ButtonTextService buttonTextService;

    @Before
    public void setupDagger() {
        final TestButtonInUiAppComponent component = buildAppComponent();
        getApp().setComponent(component);

        component.inject(this);
    }

    @Test
    public void buttonUpdatedWithText() {
        when(buttonTextService.getText()).thenReturn("Test456");

        activityRule.launchActivity(null);

        onView(withId(R.id.button)).check(matches(withText("Test456")));
    }

    private static TestButtonInUiAppComponent buildAppComponent() {
        return DaggerTestButtonInUi_TestButtonInUiAppComponent.builder()
                .mockedButtonModule(new MockedButtonModule())
                .build();
    }
}
```

[Adding DaggerMock to your project is a doddle](https://github.com/fabioCollini/DaggerMock#jitpack-configuration) and it
does away with all the aforementioned setup code. By adding in the `DaggerMockRule` and pointing it to your existing
component/modules it will automatically look for `@Mock` annotated fields then override the provider methods with a
mock, which is then set in the field for stubbing.


```java
public class TestButtonInUi extends UiTest {

    @Rule
    public DaggerMockRule<AppComponent> daggerRule = new DaggerMockRule<>(AppComponent.class,
            new AppModule(getApp()), new ButtonModule()).set(new DaggerMockRule.ComponentSetter<AppComponent>() {
        @Override
        public void setComponent(AppComponent appComponent) {
            getApp().setComponent(appComponent);
        }
    });

    @Rule
    public final ActivityTestRule<MainActivity> activityRule = new ActivityTestRule<>(MainActivity.class, false, false);

    @Mock
    private ButtonTextService buttonTextService;

    @Test
    public void buttonUpdatedWithText() {
        when(buttonTextService.getText()).thenReturn("Test123");

        activityRule.launchActivity(null);

        onView(withId(R.id.button)).check(matches(withText("Test123")));
    }
}
```

For me the ease with which I could inject mocked objects into my project was enough, but along with custom rules and
support of submodules (although limited) it also has the [`InjectFromComponent`](https://github.com/fabioCollini/DaggerMock#injectfromcomponent-annotation) which provides access
to objects injected into classes listed in the component.
