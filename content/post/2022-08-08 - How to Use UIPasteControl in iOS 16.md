+++
title = "How to Use UIPasteControl in iOS 16"
date = 2022-08-08T16:00:00-04:00
tags = ["programming"]
+++

At WWDC22, Apple announced that iOS 16 will show a modal prompt when an app tries to paste from the pasteboard without user interaction. Along with this change, Apple [introduced](https://developer.apple.com/videos/play/wwdc2022/10096/?time=564) a new UIKit control that gives apps a standard way to let users paste without showing said prompt: [`UIPasteControl`](https://developer.apple.com/documentation/uikit/uipastecontrol). At the time I wrote this, the documentation was pretty sparse. Most of the flow is based on existing APIs, but you may not be aware they exist—I wasn't! This post aims to get you started using `UIPasteControl` and its associated APIs.

If you don't care to skim this post and just want a full code example, check out [this GitHub repo](https://github.com/klanchman/UIPasteControlExample).

## Initial Setup

We need to do a couple things to start out: add a `UIPasteControl` to the screen, and hook it up to its target.

Getting the `UIPasteControl` onto the screen is the same as anything else, you can do it in Interface Builder or programmatically. I'll skip past this part 😄

Next we need to set the paste control's `target`, the object that will receive the pasted information when the control is tapped. The target needs to conform to [`UIPasteConfigurationSupporting`](https://developer.apple.com/documentation/uikit/uipasteconfigurationsupporting). All `UIResponder` subclasses do so, including `UIViewController`, so the easiest way to start is by setting the control's `target` to the view controller that it's inside of. For example:

```swift
class ViewController: UIViewController {
  private let pasteControl = UIPasteControl()

  override viewDidLoad() {
    super.viewDidLoad()

    pasteControl.target = self
  }
}
```

When we run the app right now we can see the paste control, but it is disabled (dimmed and not tappable). That's because we haven't told the system what kind of content our `target` supports having pasted.

## Set a `UIPasteConfiguration`

### Set up Supported Types

One of the properties we need to set up is [`pasteConfiguration`](https://developer.apple.com/documentation/uikit/uipasteconfigurationsupporting/2882040-pasteconfiguration). This property tells the system what kind of data the target can support having pasted. We do this by saying what Uniform Type Identifiers (UTIs) we support, or which class (that has defined UTIs) we support.

If you want to support multiple different kinds of items, you'll need to define them using UTIs. Otherwise, if you only want to support items defined by a single class (like a `String` or `UIImage`, but not both at the same time), then using a class is a bit easier to read.

#### Supporting Multiple Data Types (Using UTIs)

We can import `UniformTypeIdentifiers` to get a list of known UTIs, and then set up our `pasteConfiguration` with an array of the identifiers that we support:

```swift
import UniformTypeIdentifiers

class ViewController: UIViewController {
  // ...
  override viewDidLoad() {
    // ...

    pasteConfiguration = UIPasteConfiguration(acceptableTypeIdentifiers: [
      UTType.image.identifier,
      UTType.text.identifier,
    ])
  }
}
```

#### Supporting a Single Data Type (Using a Class)

We can set up our `pasteConfiguration` with the class that we support:

```swift
class ViewController: UIViewController {
  // ...
  override viewDidLoad() {
    // ...

    pasteConfiguration = UIPasteConfiguration(forAccepting: String.self)
  }
}
```

### Run the App Again

If we run the app now and **have a supported item on the pasteboard**, the paste control is enabled! 🎉

But if we press it, our app crashes! 💥
```text
*** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'pasteItemProviders: must be overridden if pasteConfiguration is not nil.'
```

As the exception tells us, we have a little more work to do.

## Implement `paste(itemProviders:)`

The exception says we need to implement the [`paste(itemProviders:)`](https://developer.apple.com/documentation/uikit/uipasteconfigurationsupporting/2887579-paste) method, which is where the pasted data gets sent for us to use.

We can start easily enough:

```swift
extension ViewController {
  override func paste(itemProviders: [NSItemProvider]) {
    // But what goes here? 🤔
  }
}
```

Once the user pastes, the system will give us an array of [`NSItemProvider`](https://developer.apple.com/documentation/foundation/nsitemprovider) objects containing the data that was pasted. Working with these isn't as straightforward as grabbing things from `UIPasteboard.general`, but it's not too bad after you've seen it once.

What goes inside the method will change a bit depending on whether you need to support multiple data types or just one, and how many items you want to support being pasted at once (for instance, there could be _multiple_ images being pasted). Ultimately though, this part is totally up to you and what you need to do in your app!

### A Simple Example

Let's say we only want to accept `String` data, and can only handle 1 item. Your code might look something like this:

```swift
extension ViewController {
  override func paste(itemProviders: [NSItemProvider]) {
    for provider in itemProviders {
      // This check is optional. If you don't do it, you're more likely
      // to get an error from `loadObject(ofClass:completionHandler)`.
      // You need to choose a valid UTType for whatever class you support.
      if provider.hasItemConformingToTypeIdentifier(UTType.text.identifier) {
        _ = provider.loadObject(ofClass: String.self) { str, error in
          // Ensure `error` == nil, and then do stuff with `str`
          // If you need to update the UI, switch to the main queue
        }

        // Ignore other items
        break
      }
    }
  }
}
```

### A More Complex Example

If you're accepting multiple kinds of data, like text and images, your code might look more like this:

```swift
extension ViewController {
  override func paste(itemProviders: [NSItemProvider]) {
    for provider in itemProviders {
      if provider.hasItemConformingToTypeIdentifier(UTType.text.identifier) {
        let progress = provider.loadObject(ofClass: String.self) { str, error in
          // Ensure `error` == nil, and then do stuff with `str`
          // If you need to update the UI, switch to the main queue
        }
      } else if provider.hasItemConformingToTypeIdentifier(UTType.image.identifier) {
        _ = provider.loadObject(ofClass: UIImage.self) { img, error in
          // Ensure `error` == nil, and then do stuff with `img`
          // If you need to update the UI, switch to the main queue
        }
      }
    }
  }
}
```

### Another Simple Way

**I don't recommend this approach**, but when I wrote this post (Xcode 14 beta 4) this also worked:

```swift
extension ViewController {
  override func paste(itemProviders: [NSItemProvider]) {
    let str = UIPasteboard.general.string
    // Do stuff with `str`
  }
}
```

What's happening here is we're pasting twice: once when the user presses the control, and again when the system sends us the pasted data. We ignore the data that was pasted when the user pressed the control, and instead fetch the data again from the pasteboard. This does not display the prompt since the same data was already pasted with the user's consent.

## Wrap Up

At this point, we're done! If we run the app again, put supported data onto the pasteboard, then press the paste control, our custom paste handling is executed and a prompt is not shown to the user! 🤘

## Next Steps

There are a handful things we didn't cover here that you may want to know.

To configure the UIPasteControl's appearance in code, use the `UIPasteControl(configuration:)` initializer. You pass a `UIPasteControl.Configuration` object to it, which lets you change things like the control's colors and the visibility of the icon and label.

To add a no-prompt paste button in a `UIMenu`, check out [this post by Will Bishop](https://blog.willbish.com/2022/08/02/using-uipastecontrol-in-a-uimenu-in-ios-16/). It turns out that you don't need `UIPasteControl` for this scenario!

~~If you want to put a paste control in SwiftUI, this post hopefully gave you a general starting point. You'll need to make a `UIViewRepresentable` version of UIPasteControl, and since you won't have a view controller in the mix you'll need to [make a `Coordinator`](https://developer.apple.com/documentation/swiftui/uiviewrepresentable/makecoordinator()-9e4i4) object that conforms to `UIPasteConfigurationSupporting`, and set the UIPasteControl's `target` to be the Coordinator.~~
**Update 2022-08-12:** Somehow I missed that SwiftUI has a [`PasteButton`](https://developer.apple.com/documentation/swiftui/pastebutton/) view. If you need a paste control in SwiftUI, use that! 😄

Again, you can find a simple example project [on GitHub](https://github.com/klanchman/UIPasteControlExample). Happy pasting!
