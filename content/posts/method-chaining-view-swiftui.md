+++
title = "Method Chaining for View in Swift UI"
description = """
    Take advantage of method chaining to modify a View in Swift
"""
type = ["posts","post"]
tags = [
    "swift",
    "swiftui",
    "apple",
    "ui"
]
date = "2024-11-01T00:00:00+01:00"
categories = [
    "Mobile",
    "Desktop"
]
+++
### Acknowledgements

Thanks to [@Kyokook Hwang][kyokook] who came up with the solution.

## TL;DR

```swift
struct ContentView: View {
    var action: (() -> Void)?

    var body: some View {
        Button(action: { print("hello world") })
    }

    func onAction(perform: (() -> Void)?) -> some View {
        var new = self
        new.action = perform
        return new
    }
}
```

## Introduction

When using the Swift UI API, you have probably realised there are a few ways
of passing closures to a View. One of them is passing them in the constructor,
either explicitly or using trailing closures, even multiple of them.

```swift
Button {
    print("you pressed me")
} label: {
    Image(systemName: "gear")
}
```

But there is another way deeply ingrained within Swift, which can be seen
when setting lifecycle callbacks, such as `onAppear`.

```swift
VStack {
    Text("Hello world")
}.onAppear {
    print("Text appeared")
}
```

But how can you make use of this technique yourself?


## Method chaining

My first attempt at doing this relied on method chaining, which can be used
to modify an object multiple times within the same statement.
You may have seen this technique used in Alamofire, for instance.

```swift
let r = await AF.request("https://httpbin.org/get")
                .authenticate(username: "user", password: "pass")
                .validate()
```

Let's take a look at the [source code][alamofire] to understand how this works.

```swift
@discardableResult
public func validate(_ validation: @escaping Validation) -> Self {
    /// implementation
    return self
}
```

By returning self within a method, we can call another method on the result,
and so on.

## Chaining with View

However, when you attempt to apply method chaining to a View things
get complicated. First of all, you cannot modify self without annotating
the method as `mutating`, but if you do that the compiler will let you
know that Views are **immutable**, and you cannot use mutating methods
on them.

![Views are immutable](/images/method-chaining-view-swiftui/compiler-error.png)

You may try to get around this using `@State` to hold the closure, but this
won't work either.

[@Kyokook Hwang][kyokoow] came up with a solution for this.
Instead of returning self, it is possible to create a *copy* of self,
modify it, then return it.
The only requirement is for the callback to be a public var.

```swift
struct ContentView: View {
    var action: (() -> Void)?

    func onAction(_ perform: @escaping () -> Void) -> some View {
        var new = self
        new.callback = perform
        return new
    }

    var body: some View {
        Button(action: {action})
    }
}
```

We can even go one step further and support optional callbacks without
requiring them to be wrapped in any way.
Most of the View lifecycle callbacks do this.
Another advantage of this is that you can get rid of `@escaping`


```swift
func onAction(_ perform: (() -> Void)?) -> some View {
    var new = self
    new.callback = perform
    return new
}
```

From here, you could write a protocol to enforce this interface.


## Alternative: @Environment and ViewModifier

Originally, I was going to write about this method,
until I found the technique I showed above.
Though verbose, it is also possible to use `@Environment` to set a closure,
and then use `ViewModifier` to simplify the syntax.

The only benefit of using this approach is the fact that you can use a
private var.

You can still read about that method on [StackOverflow][so-answer].


## Related material

- Apple developer documentation: [environmentkey][environmentkey]
- Stackoverflow: [Howe to implement custom callback action][so-callback]
- Stackoverflow: [How to store closure in @Environment][so-closure-env]
- Stackoverflow: [How to implement function like onAppear][so-answer]


<!-- LINKS -->
[environmentkey]: https://developer.apple.com/documentation/swiftui/environmentkey
[so-closure-env]: https://stackoverflow.com/questions/68748549/swiftui-store-closure-in-environment
[so-answer]: https://stackoverflow.com/questions/61039657/how-to-implement-function-like-onappear-in-swiftui
[so-callback]: https://stackoverflow.com/questions/64252145/how-to-implement-custom-callback-action-in-swiftui-similar-to-onappear-function
[alamofire]: https://github.com/Alamofire/Alamofire/blob/master/Source/Core/DataRequest.swift#L145
[kyokook]: https://stackoverflow.com/users/579236/kyokook-hwang
