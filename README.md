#PrefSyncService

Since an Android Wear device is basically an accessory for a phone, it’s often convenient to place the configuration UI for watch apps on the linked phone, rather than on the watch itself. But this raises a problem: how to get those user preferences from the phone to the watch?

The problem is compounded if you (the developer) also want to provide a configuration UI on the watch - perhaps for everything, or perhaps only for the most commonly-used settings. Then it’s not just a one-way data transfer, but a two-way synchronization that’s required. When the user makes a change on one UI, they expect to see it reflected on the other as well.

Neither of these problems is inherently **hard**, but under the architecture of Android Wear, they can be kind of fiddly to get right. You need to connect to the Google API client on both sides (making sure that updates don’t get lost before that connection is finalized), move your data in and out of the API, and ensure that only new data is getting sent. And if you’re adding Wear functionality to more than one phone app (to take one example) - or creating more than one Wear watch face, for another - you’ll find yourself repeating all this work every time, for every app.

Enter `PrefSyncService`, an Android helper class that aims to take the drudgery out of this task, plus ensure that you get it right every time.

At its heart, it does what it says on the tin: it’s an Android SERVICE that SYNChronizes PREFerences between different devices on the “personal area network” (_PAN_) that Wear creates. Save a value to `SharedPreferences` on one device, and if `PrefSycService` is active, it’ll send that value to every node on the PAN, and save it to `SharedPreferences` there.  Simple as that. 

So let’s get started...
## Installation
Since `PrefSyncService` is fully contained in a single Java file, simply copy it from this repository and into your Android project. You’ll need to change the package name to match your own, but Android Studio will even help you with that.
## Use
1.Install the `PrefSyncService` class (see previous section)

2.Add it to your manifest:
```xml
<service android:name=".PrefSyncService" android:enabled="true" android:exported="true">
    <intent-filter>
        <action android:name="com.google.android.gms.wearable.DATA_CHANGED" />
        <data android:scheme="wear" android:host="*"
             android:pathPrefix="/PrefSyncService/data/settings" />
    </intent-filter>
</service>
```
You may need to change `android:name` if the class isn’t in your app’s main package. And, you only need this in an app that’s receiving the data. So for instance, if your settings sync is purely from handheld to wearable, then only the wearable is receiving data - and the handheld app shouldn’t have this element in its manifest.

3.Add to `build.gradle`: (if you’re developing for Wear, you may already have this)
```
compile 'com.google.android.gms:play-services-wearable:8.3+'
```
4.Instantiate `PrefListener` and resume (/pause) wherever settings you want to sync are changed in your app. A typical case would be in a `PreferenceActivity` or configuration `Fragment`; in either one you’d instantiate a `PrefListener` in your `onCreate` method, resume in your `onStart`, and pause in your `onStop`.

5.On the receiving side (optional):
  - Add an `onSharedPreferenceChanged` listener to detect when a value has been synchronized, and act on the new data

## Optional Extensions

The usage outlined above will get you up and running with `PrefSyncService`, and is sufficient for many cases. However, there are also a few optional goodies you can elect to use if you have the need:

### `SyncFilter`
This is a simple interface that you can choose to implement to filter which prefs are synced. The prototype looks like this:
```java
public interface SyncFilter {
   boolean shouldSyncPref(String key);
}
```
 The single method simply returns `true` or `false` depending on whether the supplied `key` should be synchronized. The `PrefListener` class (see #4 above) then has a public field named `syncFilter` that you can set to the filter you’ve defined. 
Here’s a simple example:
```java
public static class LocationSyncFilter implements PrefSyncService.SyncFilter {
    // Filter on preferences which affect the device location
    @Override
    public boolean shouldSyncPref(String key) {
       switch (key) {
           case "latitude":
           case "longitude": 
               return true;
           default:
               return false;
        }
    }
}
```
And then you attach it to your `PrefListener` like so:
```java
prefListener.syncFilter = new LocationSyncFilter();
```

### Change the sync delay 
By default, `PrefSyncService` waits one second before sending any given preference change on to the Data API. The idea here is that, if you have a bunch of preferences changing in quick succession (maybe during an initialization step), they’ll be batched into a single API call for greater efficiency.
The delay is controlled by a simple static `int` in the main `PrefSyncService.java` file:
```java
public static int delayMillis = 1000;
```
If the one-second delay isn’t optimal for your needs, simply adjust this value right in the source you’ve copied into your project.

### Rename the `SharedPreferences` file
Another simple static in `PrefSyncService.java` controls the name of the `SharedPreferences` file that will be synced from.
```
public static String sharedPrefsName = null;
```
With the default value of null, `PrefSyncService` calls `PreferenceManager.getDefaultSharedPreferences` to open the preference file for syncing. If your app is using a different file, however, simply supply its full pathname to in the `String` above, and `PrefSyncService` will use that instead.

### Force resynchronization
Most of the time, `PrefSyncService` runs in the background, syncing data as changes happen. But occasionally, you might want to resync everything - and ‘PrefListener’ includes a couple of utility functions to do just that.
```java
public void resyncLocal()
```
Calling `resyncLocal` will re-synchronize all `SharedPreferences` values to the local node from any other attached nodes. It's up to the remote node(s) to decide which values to send (by implementing a `SyncFilter`).

```java
public void resyncRemote()
```
Re-synchronize all matching `SharedPreferences` values from the local node to other attached nodes. Note that a `SyncFilter` is **required** in this case; otherwise we'd synchronize ALL prefs. If this is actually what you want, use a `SyncFilter` that always returns `true`.

## Known Issues
- Calling `resyncRemote()` clears all settings from the Wear Data API as part of the resync. So if you’re using more than one `SyncFilter`, you’ll lose settings from all except the one you supplied to the `resyncRemote()` call.

## Beyond Settings
`PrefSyncService` was created to synchronize user preferences, but after using it in a couple of production projects, I came to realize it could do more. Essentially, it’s a generic data-syncing mechanism, and can be used as a wrapper for Wear’s Data API to send any sort of data between one device on the personal-area network and another.

Say for example that you want to display phone battery level on a watch face. `PrefSyncService` can serve as the backbone for this in a couple of ways:
When the watch face becomes visible, set a `SharedPreference` value on the watch to indicate this. `PrefSyncService` will sync it to the handheld, where you can have code to enable a `BroadcastReceiver` for battery-related events.
When one of these events occurs, your `BroadcastReceiver` saves the new battery level to `SharedPreferences`. `PrefSyncService` sends that level over to the wearable - where you can then display it on your watch face.
Neither of these data items being synced are _preferences_, exactly, but if you already have `PrefSyncService` in place, it’s an easy way to move the data. It also has the advantage of inherently saving state on both sides: if the connection between the watch and phone drops, for example, the last known phone battery level is already saved in `SharedPreferences` on the watch. And when the connection comes back, that value will automatically be updated.


> Written with [StackEdit](https://stackedit.io/).