---
title: AndrodUI测试入门
date: 2018-03-05 17:11:28
tags: [Android,Test]
categories: Android
---

## UI测试 
UI 测试是为了确保对于用户的UI动作，app能返回正确的UI输出。根据实际实现方案大体可以分为两种：
* End-To-End（E2E）UI测试，直接通过客户端和后台服务器的交互测试整个系统，普通操作UI，通过网络获取数据，验证UI数据。实现简单，但是存在测试速度缓慢，可能因为网络导致测试用例不通过的问题。
* 封闭UI测试，测试方法使得测试不需要外部依赖和网络请求，使用Mock Server或者其他方式替代真实的网络请求，只验证UI输出的正确性。

## UI测试框架
Android之前比较流行的UI测试框架有[robotium](https://github.com/RobotiumTech/robotium)、[Appium](http://appium.io/?yyue=a21bo.50862.201879)、[uiautomator]()、[Calabash](http://calaba.sh/?yyue=a21bo.50862.201879)、[Espresso](https://developer.android.com/training/testing/espresso/index.html),但是其中Espresso作为Google官方开源的UI测试框架，以其官方的身份、完整的使用文档以及简单的使用方法，快速成为UI测试框架中的主流，本文就是以Espresso框架为主要测试框架。

## Espresso
### 介绍及集成
Espresso 测试框架提供了一组 API 来构建 UI 测试，用于测试应用中的用户流。利用这些 API，您可以编写简洁、运行可靠的自动化 UI 测试。Espresso 非常适合编写白盒自动化测试，其中测试代码将利用所测试应用的实现代码详情。  
目前Espresso最新的版本已经出道3.0.1，使用AS创建的工程，默认已经集成了2.2.2版本的Espresso，但是如果要集成最新版本的Espresso库，需要在仓库配置中添加对应仓库地址：

    allprojects {
        repositories {
            jcenter()
            maven {
                url "https://maven.google.com"
                //Espresso3.0.1所在仓库地址
            }
        }
    }

默认集成的Espresso包espresso-core及其相关依赖包，足以完成一般性的UI测试，除此之外Espresso还有一些扩展包，用于完成一些特殊的测试场景:
* espresso-web 提供了对WebView测试的相关支持
* espresso-contrib 提供了对DatePicker, RecyclerView 和 Drawer等控件的特有动作、无障碍以及CountingIdlingResource的支持
* espresso-intents 用于校验多app测试中intent的正确性
* espresso-idling-resource（已经包含在core的依赖中）用于处理异步线程同步问题

如果测试过程中不需要上述的扩展功能，则只需要添加core的依赖

    dependencies {
        androidTestCompile('com.android.support.test.espresso:espresso-core:3.0.1', {
            exclude group: 'com.android.support', module: 'support-annotations'
            //不导入依赖中的support-annotations，避免出现依赖冲突，会使用用户自己导入的包
        })
    }
其余诸如runner，rules包都被core依赖，会自动导入，没有必要手动导入，以免导入版本不正确引起其他问题，除了上面描述的相关库，Espresso还依赖了JUnit和Hamcrest等其他测试辅助框架。
### EspressoUI测试的重要对象
* **[`Espresso`](https://developer.android.com/reference/android/support/test/espresso/Espresso.html)** Espresso框架的入口类，提供了一些静态方法，便于开始整个测试代码，它提供了类似onView和onData这种方法获取一个可交互的对象ViewInteraction，或者直接进行一个例如页面返回的顶层操作。
* **[`ViewMatchers`](https://developer.android.com/reference/android/support/test/espresso/matcher/ViewMatchers.html)** 定义了一系列静态方法用于根据不同条件返回Matcher<? super View>对象，作为参数传递给onView()。
* **[`ViewActions`](https://developer.android.com/reference/android/support/test/espresso/action/ViewActions.html)** view的操作行为例如click()，最为ViewInteraction.perform()的参数用于对指定View的进行对应操作。
* **[`ViewAssertions`](https://developer.android.com/reference/android/support/test/espresso/assertion/ViewAssertions.html)** 作为ViewInteraction.check()的参数，判断view的输出是否正确
* **[`ActivityTestRule`](https://developer.android.com/reference/android/support/test/rule/ActivityTestRule.html)** 提供了测试单个Activity的功能，当它的launchActivity设置为true时，它会在每个使用`@Test`注释的方法前和所有注释者`@Before`的方法前启动。同时可以通过ActivityTestRule对象获取对应Activity的对象。

一个简单的代码示例如下：

    @RunWith(AndroidJUnit4.class)
    public class LoginTest {
        @Rule
        public ActivityTestRule<LoginActivity> mActivityRule =
                new ActivityTestRule(LoginActivity.class);
                
        @Test
        public void login() throws Exception {
            onView(withId(R.id.et_login_number)).perform(click(),replaceText("17720380994"),closeSoftKeyboard());
            onView(withId(R.id.btn_login_next)).perform(click());
            onView(withId(R.id.et_password)).perform(click(),replaceText("aa123456"),closeSoftKeyboard());
            onView(withId(R.id.btn_login)).perform(click());
            onView(withId(R.id.toolbar)).check(matches(isDisplayed()));
            onView(allOf(instanceOf(ImageButton.class),withParent(withId(R.id.toolbar)),isDisplayed())).perform(click());
            onView(withId(R.id.tv_phone_number)).check(matches(withText("17720380994")));
            onView(IsInstanceOf.<View>instanceOf(ScrollView.class)).perform(swipeUp());
            onView(withId(R.id.tv_exit)).perform(click());
            onView(withText(R.string.exit_login_confirm)).check(matches(isDisplayed()));
            onView(withId(R.id.tv_ok)).perform(click());
            onView(withId(R.id.et_login_number)).check(matches(isDisplayed()));
        }
    }
总体来说UI测试的过程就是：**找到某个元素，做一些操作，检查结果**。

### 寻找View
Espresso中定位View主要有两种，通过页面显示的View特征（onView）和通过数据内容（onData），其中onView用于普通场景，onData用于adapterView这种可能没有渲染的view，但是两者都是基于hamcrest的matcher来进行，本质是相同的不同的是匹配规则
#### ViewMathcer
`ViewMathcer`实质上提供了很多Matcher对象，主要用于配合OnView匹配控件，这些Matcher同时可以配合hamcrest中的matcher一起使用，效果更好。常用的Matcher如下
* `withId()` `onView(withId(R.id.tv_ok))`
  直接通过id定位指定的的View，简单粗暴，但是非常实用。
* `isAssignableFrom()` `onView(isAssignableFrom(ScrollView.class))`通过对象类型判断
* `isDisplayed()`  `onView(allOf(isDisplayed(),isAssignableFrom(ScrollView.class)))` 通过是否显示判断，通常和其他matcher配合(`allOf`是hamcrest库重的方法，用于匹配多个matcher，类似的还有`anyOf`)
* `isEnabled()`
* `isFocusable()`     
  ......

`ViewMathcer`中几乎把所有的View属性都定义了对应的matcher，需要的可以自行查阅源码或文档。

#### DataInteraction
`DataInteraction` 是onData方法的返回值，因为onData方法匹配出的不直接就是View，它匹配的是一个数据集合，只有我们想要进行具体的View操作时，Espresso才会把它转化为View。
​    
     onData(instanceOf(Account.class))
Espresso没有为`onData`定义Matcher，基本都是使用hamcrest中定义的matcher或者自定义matcher

##### 自定义Matcher
一般自定义Matcher都继承`TypeSafeMatcher`，需要实现的方法如下

    public class CallInfoMatcher extends TypeSafeMatcher<CallInfo> {
        @Override
        public void describeTo(Description description) {
            //匹配失败时的描述，用于描述具体的匹配失败信息
        }
    
        @Override
        protected boolean matchesSafely(CallInfo item) {
            //具体的匹配过程
            return false;
        }
    
    }

### 对View的操作
View的操作都是在`ViewInteraction`上进行的。`ViewInteraction`也就是`onView`的返回值对象，用于对于具体的View进行操作（`DataInteraction`的操作也是转换为ViewInteraction后进行的），`ViewInteraction`提供了如下方法来对相应的元素做操作：

    public ViewInteraction perform(final ViewAction... viewActions) {}
具体的操作通过`ViewAction`定义，连续操作可以链式调用或者作为参数顺序排列。
#### ViewAction
`ViewAction`是espresso中定义的针对View操作的接口类型。`ViewAction`中实现主要在ViewActions类中通过静态方法提供。常见的action如下
* `click()`
* `closeSoftKeyboard()`
* `replaceText() `    
  ......

除去ViewActions提供的较为通用的操作方法，Espresso还提供了很多ViewAction的子类用于完成不同View的特定操作。     
> ViewAction是在View匹配成功的基础上进行的匹配失败或者匹配不唯一都会导致测试不通过，同时Action与View类型不匹配也会失败

### 校验结果
测试最重要的一步就是校验结果的正确性，`ViewInteraction`提供了`check()`方法用于校验正确性

    public ViewInteraction check(final ViewAssertion viewAssert) {
        ......
    }
和`perform()`方法类似，`check()`也是可以链式调用多次校验。
#### ViewAssertion
`ViewAssertion`是espresso中定义的用于校验View状态的接口类型，同样`ViewAssertion`也主要由`ViewAssertions`中的静态方法提供。其中主要使用的就是`matches()`方法

    public static ViewAssertion matches(final Matcher<? super View> viewMatcher) {
        return new MatchesViewAssertion(checkNotNull(viewMatcher));
    }
其中参数viewMatcher就是前面用于匹配View的`ViewMatcher`。


### 异步问题
Espresso提供了大量的同步机制，这些机制主要针对于主线层的MQ，但是Espresso对于其他的异步操作是无感知的，如果View的显示依赖于网络数据，很有可能就会导致测试用例不通过，因此需要使用前面使用的`espresso-idling-resource`来保证Espresso在异步线程的可靠性。

`espresso-idling-resource`依赖添加如下

    compile("com.android.support.test.espresso:espresso-idling-resource:3.0.1") {
        exclude module: 'support-annotations'
    }
    androidTestCompile("com.android.support.test.espresso:espresso-idling-resource:3.0.1") {
        exclude module: 'support-annotations'
    }
    //由于Espresso对与异步线程无感知，我们需要在代码中主动使用IdlingResource，因此需要使用compile依赖。

### IdlingResource
Espresso主要通过`IdlingResource`这个接口类型完成对异步资源的感知，主要方法如下

    public interface IdlingResource {
        //用于标识对于的异步资源
        public String getName();
        //返回目前资源是否可用(闲置)，
        public boolean isIdleNow();
        //Espresso会注册此回掉，需要判断资源可用时主动调用
        public void registerIdleTransitionCallback(ResourceCallback callback);
        public interface ResourceCallback {
            public void onTransitionToIdle();
        }
    }
Espresso提供了几个`IdlingResource`的实现类，可以直接使用：
* [CountingIdlingResource](https://developer.android.com/reference/android/support/test/espresso/idling/CountingIdlingResource.html) 为资源提供了一个简单的使用计数，当count为0时资源为闲置状态。
* [UriIdlingResource](https://developer.android.com/reference/android/support/test/espresso/idling/net/UriIdlingResource.html) 类似`CountingIdlingResource`,但是count为0时资源不会立即为闲置状态。
* [IdlingThreadPoolExecutor](https://developer.android.com/reference/android/support/test/espresso/idling/concurrent/IdlingThreadPoolExecutor.html)  一个有`IdlingResource`功能的`ThreadPoolExecutor `。
* [IdlingScheduledThreadPoolExecutor](https://developer.android.com/reference/android/support/test/espresso/idling/concurrent/IdlingScheduledThreadPoolExecutor.html) em.. 同上

我们借`CountingIdlingResource`来了解下`IdlingResource`的主要用法，`CountingIdlingResource`主要提供的两个共有方法供我们使用
* `increment()`计数加一
* `decrement()`计数减一，为0时调用`onTransitionToIdle()`

例如使用网络请求的场景，发起请求时`increment()`表示资源被占用，请求结束时`decrement()`，表示资源被释放。同时还需要在测试代码中注册对应资源

     IdlingRegistry.getInstance().register(idlingResource);
` IdlingResource`解决了异步代码的问题，但是依旧存在问题，我们在业务逻辑代码中创建`IdlingResource`对象，同时在需要的地方去改变它的状态，然后在测试代码中使用。这无疑是为了测试而给正常的业务代码增加了不必要的逻辑。

未完待续～