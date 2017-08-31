# iOS Widget Gotchas

All observations have been made on iOS 10 and Xcode 8. Some points may not be applicable for iOS-9-style widgets.

Additional iOS 11 observations have been added.

## Call Order at Launch

(obviously missing some within the object lifecycle, but these are the ones we feel are important)

* `viewDidLoad()`
	
	* `bounds` and `traitCollection` are not yet correctly initialized, subviews frames are completely wrong (1000x1000pt)

* `widgetActiveDisplayModeDidChange(_:withMaximumSize:)`
	
	* Called with state-preserved display mode
	* Should update the preferred content size unless unambiguously defined by constraints. Can lead to infinite calls to `viewWillTransition(to:with:)` if the preferred size was not set.
	* `traitCollection` and subview frames are still incomplete/wrong, root view's frame size is equal to the maximum size that is passed as parameter
	
* `viewWillTransition(to:with:)`
	
	* iOS 11: **not** called initially
	* iOS 10: First method to be called with the correct view bounds, including subviews. Should be used to resize everything that is not updated through auto layout (e.g., collection view item sizes and insets).
	* `traitCollection` is also complete: Compact width and regular height on iPhone (incl. 6/7 Plus in landscape), both regular on iPad. Does not seem to change during rotation.
	
* `viewWillAppear(_)`

   * iOS 11: Resize everything that is not updated through auto layout.
   * iOS 10: Should probably not be implemented in most cases.
   * May be delayed slightly, see discussion below.

## Lifecycle

* The widget UI is reloaded from scratch (i.e., a completely new instance of the VC) often
	* e.g., when:
		* going back to the homescreen and waiting for ~5s
		* going back to the homescreen and opening another app
		* scrolling the widget out of view and waiting for ~5s
		
	* Updating data in `viewDidLoad()` (and potentially in the background through `widgetPerformUpdate(completion:)`) is sufficient; should not update data in `viewWillAppear(_)`
	
* Try to load/display cached data in `viewDidLoad()` for smooth launch
	
* `viewWillAppear(_)` may be called while the widget is already visible on screen: If you scroll, it's not called until scrolling has finished
	* Anything you'd do that does not depend on the correct bounds should be done in `viewDidLoad()`

* Updates to `widgetLargestAvailableDisplayMode` are animated so you should not initialize it to one value just to set it back right afterwards, as this will cause the "Show More"/"Show Less" button to flicker for a second 

* The widget process lives significantly longer than the widget VC – it may even continue to live when the widget is hidden and when doing resource-intensive stuff while the widget is off screen. This effect may be amplified when the debugger is attached, but is generally true for non-debugging use, too.

* Although this may change over time, we observed that VCs are rarely re-used – after a few seconds off screen, the VC will be discarded, and the same widget process will create a new VC when required. However, the old VC's `dealloc`/`deinit` is not called until the process becomes active again and creates a new VC, so you should not wait for `dealloc`/`deinit` to clean up or stop unnecessary work (like streaming data from your backgrounded app to the widget).

## iPad

* iOS 11 does not have a two-column layout on iPad

* Widgets can have different widths depending on the orientation and the column which they are assigned to
	* Portrait: All widgets in one column, equal widths
	* Landscape: Widgets in two user-definable columns, left widgets are wider than right widgets
	
* Widgets in the right column *seem* to rotate with a width change
	* Instead of rotating the widget, the system creates a second instance
	* The system creates two instances at once when you assign a widget to the right column, probably for snapshotting purposes
	
## Adapting to Changes Between Compact and Expanded Display Modes

* Wait for `widgetActiveDisplayModeDidChange(_)` to be called
* Update the preferred content size
* Wait for `viewWillTransition(to:with:)` to be called and update UI accordingly

## Detecting if Device is Locked

1. Make sure [Data Protection](https://developer.apple.com/library/content/documentation/IDEs/Conceptual/AppDistributionGuide/AddingCapabilities/AddingCapabilities.html#//apple_ref/doc/uid/TP40012582-CH26-SW30) is enabled for your main app
2. The first time your main app starts, create a dummy file (let's call it `ProtectionMonitor.dummy`) in a shared group container which is accessible from both the main app and the widget
	* You can create this file with `NSFileManager.default.createFile(atPath:contents:attributes:)`
	* Make sure you pass the dictionary `{NSFileProtectionKey: NSFileProtectionComplete}` as the attributes parameter. This ensures the file is protected with Data Protection.
3. Every time your widget starts, try to read the file `ProtectionMonitor.dummy`. If it is unreadable the device is locked. Otherwise it's not. 
	* You may read the file with `Data(contentsOfFile:options:)`
	* However, every other method that reads a file from disk should be fine, too

## Other Notes

* Presented view controllers are discouraged
	* Views from presented VCs are not snapshotted, leading to weird launch behaviour
	* Only the primary VC has the `extensionContext` property set (and it's `readonly`)
	* Child view controller should be fine

* If you have more than one widget, use the `UIApplicationShortcutWidget` Info-Plist key to specify the widget (by bundle ID) which should be shown upon 3D-touching the homescreen app icon.

* Expanded display mode cannot be enforced programmatically, user must expand the widget manually
	* Display mode is automatically state-preserved and restored
	* Compact mode is fixed in height, although the exact value depends on the system font size
	* Expanded mode is variable in height (within limits, approx. the screen height)

* Scroll views cannot be scrolled with the default gestures, only programmatically

* At any time, there may be zero, one, or many VC instances within one widget process
	* Apart from the obvious VC that is visible, the system may create other instances, e.g., for snapshotting purposes, and new instances may be created when rotating or changing the order/position of widgets
	
* Asserts silently quit the widget and do not show up in the debugger – logs are probably easier to find

* Data Protection should not be enabled for widgets (do not set `com.apple.developer.default-data-protection` to a valid value in your widget's entitlements file). If you do, iOS will fail to create a snapshot of your widget after showing widgets in the lock screen. You can see a similar log entry in the console:
	```
	ImageIO: IIOImageWriteSession:111: cannot create: '/private/var/mobile/Containers/Data/PluginKitPlugin/C4C7DD84-E4FC-4507-8D77-364970936BD8/Library/Caches/com.apple.notificationcenter/Snapshots/<Widget-Identifier>-PhoneTall-359-<Locale>-UICTContentSizeCategoryL-NCWidgetDisplayModeCompact.ca/assets/image0.ktx.sb-e8f991c7-ZpFG9m'
		error = 1 (Operation not permitted)
	```
	In this case, Springboard will crash the next time your widget should be shown (as the snapshot is missing).
