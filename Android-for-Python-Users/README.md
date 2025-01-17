Android for Python Users
========================

*An unofficial Users' Guide*

Revised 2021/06/05

# Introduction

Python and Kivy are portable across operating systems because of POSIX, Python wheels, and pip. However Android is not POSIX compliant, wheels are not usually available for Android, and pip is not installed on Android. Clearly many apps won't 'just work' on Android. The document is about porting a Kivy app to Android.

For a well written app that only paints the screen, and does nothing else, building for Android will be 'push button'. One can for example build 'Hello World' with the default 'buildozer.spec' file. Of course this does not describe many apps.

This document is not a substitute for Reading The Fine Manual [(RTFM)](https://en.wikipedia.org/wiki/RTFM). Your basic resources are [Buildozer](https://github.com/kivy/buildozer/tree/master/docs/source), [Python for Android](https://github.com/kivy/python-for-android/tree/develop/doc/source), and [Android](https://developer.android.com/guide).

We assume android.api >= 29

Python-for-Android is a truly amazing achievement, but there are some details to understand. The alternative is Java and the Android api, which has been shown to induce insanity in laboratory mice. 

Lastly this document is my understanding as of the date above. I am not a developer in any of the Kivy sub projects (except for one PR in p4a). The document is guaranteed to be incomplete is some way, and may possibly be wrong in another way. But reading it will hopefully provide you with come context with which to read the official documentation and to experiment. Those two steps are the key to insight and understanding, which in turn is key to making your idea real.

# What is Different about Android?

## Posix

   Android is not POSIX compliant (POSIX is so deep in your assumptions about computers you probably don't know it exists).
   
The file system model is different, the app cannot use any directory in the file system. The app has specific private storage directories that support file operations. The app also has storage shared between apps, this is implemented as a database.

Threads are available and unlike the desktop, run on all available cores. This is cool. But a subprocess can't execute an app, and there is no command shell to run inside a subprocess.

Multi-tasking; on the desktop when an app loses focus or is minimized it continues to execute - it just doesnt see UI events. Android is not multi-tasking, when an app is removed from the UI it *pauses*, and then does not execute any app code. Execution without a UI requires an Android service.

## Wheels

Some Python packages are not written in Python (are not pure-Python), they contain code that must be compiled. Pip provides pre-compiled packages for desktop OSes, but not for Android. P4a addresses this with [recipes](https://github.com/kivy/python-for-android/tree/develop/pythonforandroid/recipes), but not all impure packages are available. AVOID DISAPPOINTMENT, check availability first. [See also.](#requirements)

## Meta-information

Unlike the desktop you must provide information *about* your Python code, this requirement causes everybody who doesn't understand it to crash and burn. This information must be supplied in the buildozer.spec file. It includes *all* the pip packages your app depends on, any file types your app depends on for data, and the Android permissions your app depends on. Meta-information may not be limited to that short list, but that list is critical.

# Android Storage

Android's view of its file system has changed a few times over the years. The result is a messy pile of jargon: local, system, internal, external, primary, secondary, scoped, all describe views of storage.

However in the storage model described here storage is either *Private Storage* or *Shared Storage*. Private storage content is only visible to the app that created it, shared storage is visible to all apps. This model is valid if the build api>=29.

Physically storage may be 'internal' or 'external', this is a somewhat historical view where 'external' represents the (on older devices possibly removable) sdcard. The apis referenced here all default to 'external', with 'internal' as an option.

## Private Storage

An app can perform Python file operations (read, write, shutil) on its private storage. Files in Private Storage are persistent over the life of the app, they are retained when the app is updated, and deleted when the app is uninstalled.

The location of an app's private storage is given by for example `app_storage_path()` imported as `from android.storage import app_storage_path`

The install directory './' is also private storage, but files do not persist between installs and updates. Only use this directory for reading data files packaged in the apk.

Do not confuse private with secure, on older Android versions it is possible for other apps to read private storage. In this case internal storage is more secure than external storage.

No permissions are requires to read or write an app's private storage. [More on permissions below](#app-permissions)

## Shared Storage

Shared storage is visible to all apps, is persistent after an app is uninstalled. On devices running Android 10 and later shared storage is implemented as a database, you cannot access shared storage with POSIX file operations. The Kivy and KivyMD file choosers do not work, you must use the Android file picker.

The nearby [storage example](https://github.com/RobertFlatt/Android-for-Python/tree/main/storage) demonstrates the database insert(), delete(), and retrieve() operations.

Shared storage is organized based on root directories located in Android's 'Main Storage'. They are 'Music', 'Movies', 'Pictures', 'Documents', and 'Downloads'.

On devices running Android 10 and later no permissions are requires to read or write an app's own shared storage. Reading another app's shared storage requires READ_EXTERNAL_STORAGE permission.

On devices running Android 9 and less, for shared storage, both file access and database access are available, though the database implementation is perhaps too basic. The best choice depends on the application. The database approach enables a file share with other apps (see next section), the file system approach provides the user with the familiar hierarchy, and requires WRITE_EXTERNAL_STORAGE permission.

## Sharing a file between apps

Files in Shared Storage can be shared between apps. Either using the file picker, or the Share mechanism. See the examples: [Sending a Share](https://github.com/RobertFlatt/Android-for-Python/tree/main/share_snd) and [Receiving a Share](https://github.com/RobertFlatt/Android-for-Python/tree/main/share_rcv).

# Threads and Subprocesses

Threads are available and are have more utility than on the desktop because they are executed on all available cores. The single core thread implementation on desktops allows lazy assumptions about thread result ordering. These will bite you on Android if the code is not written in a thread safe manner. Always use callbacks.

Kivy executes on the 'UI thread', Android requires that this thread is always responsive to UI events. As a consequence long latency operations (e.g. network access, sleep()) or computationally expensive operations must be performed in their own Python threads. Threads must be truly asynchronous to the UI thread, so do not join() in the UI thread. A non-UI thread may not write to a UI widget. A very thread safe way to return results to the UI thread is to use Clock_schedule_once().

The Python subprocess is not available. The Android equivalent is an Activity, an Activity with no UI is a Service. An Activity can be created through Pyjnius and Java, by first creating an Intent. The special case of creating an Android Service can be automated using Buildozer.

# Android Service

An [Android Service](https://developer.android.com/guide/components/services) is somewhat equivalent to a Python subprocess, in that it can perform operations in the background. An Android service has asynchronous execution and an independent memory space, however unlike a subprocess it does not execute on a different core. In this context we create an Android service that can emulate a Python subprocess, this emulation does not have the lifetime of a true Android service.

The best (and only) Kivy example is [Kivy Service Osc](https://github.com/tshirtman/kivy_service_osc). OSC is a good package for message passing between app and service. However it is not designed for passing large datas, consider using the file system in this case.

## Specifying a Service 

If the context of Kivy, an Android service is a python script. The script has a file name and we give the service some name; these are associated in `buildozer.spec` (here `the_service.py` is the name of the script, and `Whatever` the name we give the service). To start a background service:
```
# (list) List of service to declare
services = Whatever:the_service.py
```

We start the service from an app using the full app package name, '.Service', and the name of the service:
```
from android import mActivity
context =  mActivity.getApplicationContext()
SERVICE_NAME = str(context.getPackageName()) + '.Service' + 'Whatever'
self.service = autoclass(SERVICE_NAME)
self.service.start(mActivity,'')
```

A [foreground service](https://developer.android.com/guide/components/foreground-services) is specified in buildozer.spec with: 
```
# (list) List of service to declare
services = Whatever:the_service.py:foreground

# (list) Permissions
android.permissions = FOREGROUND_SERVICE
```
A notification icon will be created in the task bar. 

## Service Lifetime

The lifetime of the service's Python script is usually determined by an infinite loop, if this is the case the lifetime of the service is determined by the lifetime of the app. A service started by a Kivy app executes while the app is either in the foreground or paused, and like a Python subprocess stops when the app stops. An app stops when it is removed from the list of currently started apps. This lifetime is different from the lifetime of a Java service which is persistent.

A service can be killed at any time by Android if it requires the resources. This is unlikely but possible. Extend the lifetime using [`setAutoRestartService()`](https://github.com/kivy/python-for-android/blob/develop/doc/source/services.rst#service-auto-restart) which forces a service to restart when it stops (set to False when comanding the service to stop). If you use auto restart you must use `kivy==master`. With [kivy==2.0.0](https://github.com/kivy/kivy/pull/7508), the Kivy app which started the service and which has since stopped will hang with a black screen when it is restarted and the service is still running.

## Service Performance

Unlike a Python subprocess an Android Service executes on the same core as the UI that started it (perhaps multitasking?), so the service can impact UI performance. Computational or I/O tasks in the service should be performed in threads.

## p4a Service Implementation

If you want to understand the implementation of services in more detail, read the code: [PythonService.java](https://github.com/kivy/python-for-android/blob/develop/pythonforandroid/bootstraps/common/build/src/main/java/org/kivy/android/PythonService.java), [Service.tmpl.java](https://github.com/kivy/python-for-android/blob/develop/pythonforandroid/bootstraps/common/build/templates/Service.tmpl.java), and [build.py](https://github.com/kivy/python-for-android/blob/develop/pythonforandroid/bootstraps/common/build/build.py).

# App Permissions

Android restricts access to many features. An app must declare the permissions it requires. There are two different declarations, manifest and user. User permissions are a subset of manifest permissions.

Manifest permissions are declared in the buildozer.spec file. Common examples are  CAMERA, INTERNET, READ_EXTERNAL_STORAGE, RECORD_AUDIO. The [full list is](https://developer.android.com/reference/android/Manifest.permission). Apps that scan Bluetooth or scan Wifi may require multiple permissions. In general you must research the permissions needed, resist the temptation to blindly guess.

Python for Android always enables manifest WRITE_EXTERNAL_STORAGE permission. WRITE_EXTERNAL_STORAGE implies READ_EXTERNAL_STORAGE. [WRITE_EXTERNAL_STORAGE is never required](https://developer.android.com/training/data-storage#permissions) for devices running api >= 30.

Any app manifest permission documented in that list as having "Protection level: dangerous" additionally require a user permission. The four listed above are all "dangerous". User permissions generate the dialog that Android apps present to the user. In a Kivy App this must be called from the `build()` method using `request_permissions()` which is part of the `android.permissions` package.

See any of the nearby examples.

# Buildozer and p4a

## Install
[RTFM](https://github.com/kivy/buildozer/blob/master/docs/source/installation.rst), really. Errors during a Buildozer build are usually because the user:

* Failed to [read the install instructions](https://github.com/kivy/buildozer/blob/master/docs/source/installation.rst), and so failed to install a dependency.

* Failed to accept the Google License Agreements.

* Attempted to build an impure Python package.

Buildozer runs on Linux, Windows users need a Linux virtual machine such as WSL, Colab, or VirtualBox to run Buildozer.

Buildozer's behavior can be non-deterministic in any of these cases:

* It is run as root

* It is run on an NTFS partition mounted on a Linux system.

* There are Python style trailing comments in the buildozer.spec


## Changing buildozer.spec

Note that Buildozer allows *specification* of build files, versions, and options; but unlike most other build tools it *does not do version management*. If buildozer.spec is changed the change probably *won't* propagate into the apk on the next build. After changing the buildozer.spec file users *must* do an appclean.
```
buildozer appclean
buildozer android debug
```
There may be some exceptions to this, the only one I know to be safe is one can add (but not change version of, or remove) a package in the [requirements](#requirements) list without the appclean.

There is no magic universal buildozer.spec, its configuration depends on the functionality of your app. 

## Some buildozer.spec options

[RTFM](https://github.com/kivy/buildozer/blob/master/docs/source/specifications.rst), really. And see the [KivyMD section](#kivymd).

### package.domain

This must contain exctly one period (.) surrounded by alpha numeric characters, do not use any other special characters. Anthing else will cause a Gradle fail.

### requirements

#### requirements basics

This is basically the list of pip packages and the Python version that your app depends on. On the desktop this is handled for you by `pip3 install`, at the cost of disk space. On Android there is no `pip3` so you have to do it by hand.

Generally you can find what packages a package depends on by looking on GitHub at the package's `requirements.txt` file (if it exists). These packages must be added to requirements in buildozer.spec.   

It is important that you understand what your app depends on. The requirements list must be complete, missing one item is the most common cause of a run time crash on Android. The next most common cause is adding a package that is not pure Python or does not have a recipe.

Do not add Python system modules, only packages you might install with pip3 on the desktop. Or for some recipes, other recipies whose name begins with 'lib'.

There are some pip3 packages that are added automatically, no need to put these in requirements: `libffi, openssl, sqlite3, setuptools, six, pyjnius, android, certifi`.

The current Buildozer default version for Kivy is obsolete, change it to 
`requirements = python3,kivy==2.0.0`

The packages you add here **must be pure Python, or have a recipe** [in this list](https://github.com/kivy/python-for-android/tree/develop/pythonforandroid/recipes). If this is not the case, the options are to:

* Rewrite the app

* Locally modify an existing recipe [see Appendix C](#appendix-c--locally-modifying-a-recipe).

* [Create a new recipe](https://github.com/kivy/python-for-android/blob/develop/doc/source/recipes.rst).

* Import the functionality from Java.

None of these options are trivial. That is why it said AVOID DISAPPOINTMENT in [the Wheels section above](#wheels).

If you have a problem run the [debugger](#debugging).

#### Simple requirements examples

Some packages have dependencies but no requirements.txt file, the only way to resolve these is with the debugger. One example:

`import requests` needs `requirements = python3,kivy==2.0.0,requests,urllib3,chardet,idna`

Some Kivy widgets depend on other packages. For example:

`from kivy.uix.videoplayer import VideoPlayer` needs `requirements = python3,kivy==2.0.0,ffpyplayer`

`from kivy.core.audio import SoundLoader`  needs `requirements = python3,kivy==2.0.0,ffpyplayer,ffpyplayer_codecs` [to play .mp3](https://github.com/Sahil-pixel/kivy-with-mp3-on-android-).

Some pip3 package names are not the same as the class name. For example:

`from bs4 import BeautifulSoup` needs `requirements = python3,kivy==2.0.0,beautifulsoup4`

Some recipe names are not the same as the class name. For example:

`import google.protobuf` needs `requirements = python3,kivy==2.0.0,protobuf_cpp`

#### More complex requirements

Some imports have more than one of the above cases. To determine a package's dependencies look in requirements.txt recursively. For example for `pyrebase` start with [requirements.txt](https://github.com/thisbejim/Pyrebase/blob/master/requirements.txt) to see the dependencies, there are six. The first is `requests`, this one is easy because its dependencies are listed earlier in this section. For the others recur. If you miss one it will show up as a `ModuleNotFoundError` at run time. It is not hard, just stop whining and do the work. For example for `pyrebase` we get:

`import pyrebase` needs `requirements = python3, kivy==2.0.0, pyrebase, requests, urllib3, chardet, idna, gcloud, oauth2client, requests-toolbelt, protobuf_cpp, python-jwt, pycryptodome, httplib2, pyparsing, pyasn1, pyasn1_modules, rsa, jwcrypto, cryptography`

Anybody got any more examples I could add here?

### source.include_exts

If the app contains any data files, for example icons. Then add the file extension to this list:
```
source.include_exts = py,png,jpg,kv,atlas
```

### android.permissions

Research the Android permissions your app needs. For example
```
android.permissions = INTERNET, CAMERA, READ_EXTERNAL_STORAGE
```

### android.api

The current buildozer default is 27, but should be "as high as possible". 
```
android.api = 30
```

### android.minapi

Python for Android enables `android.minapi = 21`.

### android.ndk

Probably best not to change this from the current 19c. But if there is some reason you really need to, 21d also appears to work.

### android.arch

Currently defaults to 32-bit ARM. For performance improvement, if you have a 64 bit device, change this to:
```
android.arch = arm64-v8a
```
An install message INSTALL_FAILED_NO_MATCHING_ABIS means the apk was built for a different architecture than the phone or emulator.

# Debugging

On the desktop your friends are the Python stack trace, and logging or print statements. It is no different on Android. To get these we [run the debugger](https://kivy.org/doc/stable/guide/android.html#debugging-your-application-on-the-android-platform).

First connect the device via USB, on the Android device enable 'Developer Mode' and 'USB debugging'.

If Buildozer was run on a virtual machine then it may not be able to use the the physical USB port and the 'deploy run logcat' options will not work. [In this case use adb instead.](#appendix-a--using-adb)

If Buildozer was run on a WSL then it may not be able to use the the physical USB port and the 'debug deploy run logcat' options will not work. [In this case use adb instead.](#appendix-d--debugging-on-wsl)

Successful setup is indicated by log output similar to:
```
List of devices attached
0A052FDE40019P  device
```
If 'List of devices attached' is followed by an empty line then the connection failed, regardless of what the Buildozer log say afterwards. Either because the device debug options are not set or the debugger is run from a virtual machine.

In the logcat output look for 'Traceback', what follows is a Python stack trace, which usually indicates the cause of the issue. For example:
```
ModuleNotFoundError: No module named 'some-import-name'
```
Where 'some-import-name' is in 'some-pip-package', then this pip package name is missing from [buildozer.spec requirements](#requirements).

It is possible to [debug using an emulator](#appendix-b--using-an-emulator) but this is not recomended initially, as it adds unknowns to the debug process. The emulator is useful for checking a debugged app on various devices and Android versions.

# Some Related Topics

## Layout

A portable layout is not an Android specific issue. In general for Kivy apps use [density-independent pixels](https://kivy.org/doc/stable/api-kivy.metrics.html) 'dp' to specify an absolute size, and scale-independent pixels 'sp' to specify a font.

## KivyMD

The KivyMD widgets have the look and feel that Android users expect, but the Material Design rules mean you don't have the same flexibility as Kivy widgets.

KivyMD is in development, which means some functionality [is still changing](https://kivymd.readthedocs.io/en/latest/changelog/index.html). Next time KivyMD is downloaded the version number may be the same, but some widget may be different!

[How to use KivyMD with Buildozer](https://github.com/kivymd/KivyMD/blob/master/README.md#how-to-use-with-buildozer). There may be additional Buildozer settings required for KivyMD, see KivyMD's sample [buildozer.spec](https://github.com/kivymd/KivyMD/blob/master/demos/kitchen_sink/buildozer.spec).

## Camera

It is hard to get the Kivy Camera widget to work on Android, the OpenCV camera does not work on Android. Try the [Xcamera widget](https://github.com/kivy-garden/xcamera) from the Kivy Garden, it still has issues but is currently the best choice for an camera preview as part of a layout. Also there is [Color blind camera](https://github.com/inclement/colour-blind-camera) and [zbarcam](https://github.com/kivy-garden/zbarcam). Another option is [CameraXF](https://github.com/RobertFlatt/Android-for-Python/tree/main/cameraxf), a turnkey full screen photo, video, and image analysis camera.

## Keyboard

The relationship between the Android keyboard and the layout is somewhat configurable with [softinput_mode](https://kivy.org/doc/stable/api-kivy.core.window.html#kivy.core.window.WindowBase.softinput_mode), consider `Window.softinput_mode = 'below_target'`.

The type of keyboard can be set with `input_type` for example `TextInput(input_type = 'tel')`. Accepted values are 'text', 'number', 'url', 'mail', 'datetime', 'tel', or 'address'. Input_type requires `kivy==master` and does not work with kivy==2.0.0 .

## Back Button and Gesture

A back button/gesture can be detected with a test for key == 27, as shown in the [documentation](https://github.com/kivy/python-for-android/blob/develop/doc/source/apis.rst#handling-the-back-button). The key handler **must** return a boolean.

Android 10 and up require that the back button/gesture can return the app to the Android home screen, therefore there must be a state in the app where back button/gesture is not consumed by the app. For example if the back button/gesture transitions between Kivy screens, from the "main" screen the app can on a back event go to the Android home screen:
```
class Main(ScreenManager):
    def __init__(self, **kwargs): 
        super().__init__(**kwargs)
        Window.bind(on_keyboard = self.keyboard)
        
    def keyboard(self,window,key,*args):
        if key == 27 and self.current != "main":
            self.current = some_previous_screen
            return True   # key event consumed by app
        else:           
            return False  # key event passed to Android
```

If `kivy==master` is used `p4a.branch = develop` **must** be used, else there will be a run time error `object has no attribute 'changeKeyboard'`. `kivy==2.0.0` does not appear sensitive to `p4a.branch`.

## Kivy Lifecycle

Follow the [Kivy Lifecycle](https://kivy.org/doc/stable/guide/basic.html#kivy-app-life-cycle), it abstracts the app behavior that Android expects.

Do not place code in the app that interacts with Android 'script style', to be executed before the Kivy build() call.

`request_permissions()` must only be called from the App's `build()` method, and only one once with an argument that is a list of all required permissions.

The App's `on_stop()` method is not always called, use `on_pause()` to save state.

## The Android package

P4a provides Android specific utilities in the android package, this is only available on Android. It is as they say 'self documenting', which really means there isn't any documentation. [Read the code](https://github.com/kivy/python-for-android/tree/develop/pythonforandroid/recipes/android/src/android).

## Plyer

Plyer is an OS independent api for some non-POSIX OS features. See [available features](https://github.com/kivy/plyer#supported-apis).

The [Plyer examples](https://github.com/kivy/plyer/tree/master/examples) are the documentation. Some Plyer examples work on Android but not all Android versions, for example Camera, Speech to text, Storage. Also some run time permissions are missing.

If you plan to use Plyer, and the idea of it is very appealing, first try a small test case on your target Android versions. If it does what you expect, check that all the features you need are available; as Plyer has a platform lowest common denominator design.


## Pyjnius

### Basic Pyjnius Usage

[Pyjnus](https://github.com/kivy/pyjnius/tree/master/docs/source) allows import of Java code into Python code. It is an interface to the Java api and the Android api. The Android api is only available on Android devices, Android api calls must be debugged on Android.

```python
from jnius import autoclass
# declare a Java Class
DownloadManager = autoclass('android.app.DownloadManager')
# Java sub classes are delimited by a '$' in place of a '.'
DownloadManagerRequest = autoclass('android.app.DownloadManager$Request')  

   # get a class constant
   visibility = DownloadManagerRequest.VISIBILITY_VISIBLE_NOTIFY_COMPLETED
   # instance a Java class
   self.request = DownloadManagerRequest(uri)
   # call a method in that class
   self.request.setNotificationVisibility(visibility)
```

Then use this to write code with Python syntax and semantics, and Java class semantics added. Some basic knowledge of Java semantics is required, get over it. Android classes will require (possibly extensive) reading of the [Android Developer Guide](https://developer.android.com/guide) and [Android Reference](https://developer.android.com/reference).

It is also possible to write Java class implementations in Python usiny `PythonJavaClass`, [RTFM](https://github.com/kivy/pyjnius/blob/master/docs/source/api.rst#java-class-implementation-in-python) and [look at some examples](https://github.com/RobertFlatt/Android-for-Python/blob/main/cameraxf/cameraxf/listeners.py).

Note: some documentation examples are obsolete. If you see '.renpy.' as a sub field in an autoclass argument replace it with '.kivy.'.

### Pyjnius Challenges

It is not possible to import Java *abstract* classes, as they have no *implementation* (abstract and implementation are Java keywords). And it it is not possible to provide the implementation in Python. You must find or write the implementation in Java and import that new class.

One thing to watch out for is you will be using two garbage collectors working on the same heap, but they don't know each other's boundaries. Python may free a local reference to a Java object because it cant see that the object is used. Obviously this will cause the app to crash in an ugly way. So use class variables, as in the example above, to indicate persistence to the Python garbage collector.

Python for Android builds an apk with a minimum device api. Importing Java modules can invalidate this minimum. Check the [Added in API level field](https://developer.android.com/reference/android/provider/MediaStore.Downloads) in the class or method reference documentation.

### Callbacks with a Java Abstract Class.

Pyjnius allows us to call Java from Python, calling Python from Java is achived by overriding the Java class implementation in Python using `PythonJavaClass`. However we cannot do this if the Java class is abstract, in this case we implement the derived class in Java and pass the Python callback inside a Java wrapper class with a `PythonJavaClass` implememntation.

The steps are outlined below, with links to an example in cameraxf.

- Create a [Java wrapper interface class](https://github.com/RobertFlatt/Android-for-Python/blob/main/cameraxf/cameraxf/camerax_src/org/kivy/camerax/CallbackWrapper.java), the class contains the Java callback method prototypes.

- Create a [Python implementation](https://github.com/RobertFlatt/Android-for-Python/blob/main/cameraxf/cameraxf/listeners.py#L118) of the wrapper class as a subclass of `PythonJavaClass`. Note that the Python class has the Python callback method as an initialization parameter, the Java interface did not have this. 

- Create the [listener derived from the abstract class](https://github.com/RobertFlatt/Android-for-Python/blob/main/cameraxf/cameraxf/camerax_src/org/kivy/camerax/ImageSavedCallback.java) as usual. Except it has the wrapper class as an initialization parameter, so it can call the wrapper's callback method(s).

- [In Python](https://github.com/RobertFlatt/Android-for-Python/blob/main/cameraxf/cameraxf/camerax.py#L354-L356) instantiate the wrapper with a reference to the Python callback method as an parameter. In Python instantiate the listener implementing the abstract class with a reference to the wrapper as an parameter. In Python instantiate the Java functionality with it's listener as a parameter.

No, I won't do it for you.


## Kivy Garden

[Kivy Garden](https://github.com/kivy-garden/) is a library of components ('flowers'). It is mostly not maintained. Anybody who has had a garden knows a garden needs a gardener, Kivy Garden doesn't have one. Kivy Garden is a useful resource, it is mostly best used as examples to copy and modify (fix) rather than components you can install and instantiate.

## Android for Python

[Android for Python](https://github.com/RobertFlatt/Android-for-Python) contains examples of some Android features as used from Python (and this document). These examples only run on Android.

## Other Resources
[https://github.com/yunus-ceyhan](https://github.com/yunus-ceyhan)

[https://github.com/adywizard](https://github.com/adywizard)

[https://github.com/tshirtman](https://github.com/tshirtman)

## Android Store

### How to create a Release APK

[The instructions are here](https://github.com/kivy/kivy/wiki/Creating-a-Release-APK) but don't just follow the instructions, read all the annotated comments by HeRo002. The instructions are flawed, but in combination with the comments they are good.

Always build with the latest api and arm64-v8a when building for the Store.

Apparently one can upload an armeabi-v7a apk to the play store, but you must first upload an arm64-v8a version of the app.

HeRo002 tells us what to [expect](https://github.com/kivy/buildozer/issues/1290).

### How to create a Release Bundle (.aab)

The store will soon require that apps are submitted as an [app bundle](https://developer.android.com/guide/app-bundle).

Buildozer does not currently build .aab files. The following is a summary of a workaround posted on [Google Groups.](https://groups.google.com/g/kivy-users/c/LmoegwYuEEk/m/II-wuO-nAgAJ) This workaround only applies to single architecture bundles, there is no workaround for multi architecture bundles. [Because architecture specific Kivy app code is in the apk in `assets/private.mp3`, but in .aab `assets` are architecture independent. A resolution would require renaming private.mp3 for each archicture, and the app picking the right one for the device.] 

Basically the workaround specifies part of `project/.buildozer` as an Android Studio project, and lets Android Studio do the work (virtual machine users, also read the two extra items below):

1) In Android Studio `File->Open <project>/.buildozer/android/platform/build-arm64-v8a/dists/<app_name>__arm64-v8a/`

2) If Android Studio complains about a Build Tools version edit `<app_name>__arm64-v8/build.gradle` and change `BuildToolsVersion 31.0.0-rc2`  to `31.0.0.rc3` (or whatever is current).

3) If Android Studio offers a Gradle plugin update (perhaps to `4.2.0`) then accept this update. This results in using Gradle 6.7.1 (in my case).

4) Go to `Build->Generate Signed Bundle or APK`, select `Bundle` and follow the steps.


If Buildozer was run on a virtual machine:

1) copy `<project>/.buildozer/android/platform/build-arm64-v8a/dists/<app_name>__arm64-v8a/` to a desktop and open that in Android Studio.

2) Android Studio will tell you the SDK tools path is wrong, and ask to fix it; do this.


# Appendix A : Using adb

The easiest way to get adb is to install Android Studio.

Add something like this to your PATH:
     C:\Users\UserName\AppData\Local\Android\Sdk\platform-tools

Some adb commands:
```
adb devices
adb install -r  cameraxf-0.1-arm64-v8a-debug.apk
adb logcat -c
adb logcat > log.txt
```

# Appendix B : Using an emulator

Install Android Studio.

In Android Studio, go to Tools->AVD Manager

Use the 'Create Virtual Device..' button if the emulator you want is not listed.

Right click on an emulator (view details) to see it's name, for example: Nexus_4_API_21

Add something like this to your PATH:
     C:\Users\UserName\AppData\Local\Android\Sdk\emulator

Start an emulator using it's name
```
emulator @Nexus_4_API_21
```
Check the emulator is running using 'adb devices'. Then install an app in the emulator from adb. The apk MUST be built with android.arch set to the same as ABI above.

# Appendix C : Locally modifying a recipe

Modify an existing recipe by making a local copy.
Replace RECIPE_NAME with whatever recipe you are changing:

1) in buildozer.spec set `p4a.local_recipes =  ./p4a-recipes`

2) `mkdir ./p4a-recipes`

3) [Download python-for-android](https://github.com/kivy/python-for-android)

4) Copy the recipe you want from python-for-android/tree/develop/pythonforandroid/recipes/RECIPE_NAME to ./p4a-recipes

5) Change the files in a way that makes you happy

6) buildozer appclean

7) buildozer android debug


# Appendix D : Debugging on WSL

The easiest way to get adb is to install Android Studio for both Linux Subsystem (WSL) and Windows of same *Android Debug Bridge version*, use `adb --version` command to find it out


Windows setup:

Add something like this to your PATH in Windows:
    ` C:\Users\UserName\AppData\Local\Android\Sdk\platform-tools`

In cmd / pwsh, run this command and this is your server and should not be closed until you finish debugging.
```
adb kill-server
adb -a -P 5037 nodaemon server
```
Add the below environmental variable to the ~/.bashrc or ~/.zshrc , depending on your shell

Linux Subsystem (WSL) setup:
```
export ADB_SERVER_SOCKET=tcp:192.168.1.250:5037
```
Note: `192.168.1.250` is local ip of your machine find it using `ipconfig` in cmd

Finally, `adb devices` in Linux Subsystem should show your device
```
List of devices attached
0A052FDE40019P  device
```
