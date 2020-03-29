---
title: "Small example of async GUI code with Gtk-rs: alert() function"
slug: gtkrs-async-alert-example
description: null
date: 2020-03-29T20:29:45+03:00
type: posts
draft: false
categories:
- General
tags:
-
series:
-
---

*Note*: this post was written assuming the following versions:
*   Gtk-rs 0.8.1
*   Async-std 1.5.0
*   Glib-rs 0.9.3
*   futures 0.3.4

Let's suppose that I want to write the following function in Rust with Gtk-rs:

```rust
/// Display the message in the dialog window and wait for it to be dismissed by the user.
pub async fn alert(title: &str, text: &str);
```

After some experimentation, I found that one good way to implement such a function is to use one-shot channels.
Currently, `futures` crate [provides them](https://docs.rs/futures/0.3.4/futures/channel/oneshot/index.html)
on stable releases --- so I'll use those. Note that Rust's async ecosystem still evolves and things might change in the future.

Oneshot channels are created with [`oneshot::channel()`](https://docs.rs/futures/0.3.4/futures/channel/oneshot/fn.channel.html) function
and have two ends: sender and receiver.
The key property is that *the receiver end is a `Future`*. We will use this to build our functions.

In this article, I'll walk us through the implementation of `alert()` function.

That's a function epilogue and some `use`'s which we'll need later. Note that the function is declared as async
which means that it actually returns an anonymous implementation of `Future<Output=()>`.

```rust
/// Display the message in the dialog window and wait for it to be dismissed by the user.
pub async fn alert(title: &str, text: &str) {

    use gtk::prelude::*;
    use std::rc::Rc;
    use std::cell::RefCell;
    use futures::channel::oneshot::channel as oneshot_channel;
```

We will need a channel that will hold the result of our function. Since the result is an empty unit value,
this channel will act as an event or barrier on which further code will wait.

```rust
    let (result_sender, result_receiver) = oneshot_channel();
```

Next, we will have to deal with some memory management details. Here are a few important facts:
*   [`oneshot::Sender::send()`](https://docs.rs/futures/0.3.4/futures/channel/oneshot/struct.Sender.html#method.send) method consumes `Self`
*   Gtk-rs's accepts bare `Fn(&Self)` functions as callbacks
    (e.g., see [DialogExt.connect_response](https://docs.rs/gtk/0.8.1/gtk/trait.DialogExt.html#tymethod.connect_response)).
    These may be potentially called multiple times and hence they capture their environments as immutable `&`-borrows.

These two facts combined together means that we need to wrap `result_sender` in several wrappers:
*   `Option`. We will use `Option::take()` method to be able to consume the sender
*   `RefCell`. Since our callback is an `Fn()`, the captured environment is immutable and we need interior mutability to be able to send into the channel.
*   `Rc`. `result_sender` will be shared between our main async stack frame and callback.

```rust
    let result_sender = Rc::new(RefCell::new(Some(result_sender)));
```

Then we are ready to build our UI widget.

```rust
    let dialog = gtk::MessageDialogBuilder::new()
        .buttons(gtk::ButtonsType::Ok)
        .message_type(gtk::MessageType::Info)
        .text(text)
        .modal(true)
        .title(title)
        .type_(gtk::WindowType::Toplevel)
        .build();
```

Then we should attach handler for [`response` signal](https://developer.gnome.org/gtk3/stable/GtkDialog.html#GtkDialog-response) which is emitted
when user responds to dialog (e.g., by closing it or by click on `OK` button). In our case, we don't care what was the user's response - we just
need to known when user dismissed the dialog.

The `Rc` with the sender is cloned; in this case, it's not strictly necessary, but in general, it is usually required to clone objects that go
into callback's captured environment.

```rust
    let result_sender2 = result_sender.clone();
    dialog.connect_response(move |_, response| {
```

In the `response` signal handler, we need to notify the channel that we are done. First we unwrap all layers of wrappers and then send the unit value into the channel.

```rust
        println!("Response={}", response);
        let result_sender = result_sender2.borrow_mut().take().unwrap();
        result_sender.send(()).unwrap();
    });
```

After the signal handler is connected, we should show the dialog.

``` rust
    dialog.show_all();
```

And then we await from the receiver end of the channel. Since we are awaiting *asynchronously*, Gtk+ event loop keeps processing events *concurrently* while we are waiting.
And that's very important --- otherwise, our dialog which we just created would not be able to show itself on a screen and accept user input.

```rust
    result_receiver.await.unwrap();
```

After the `response` signal, the dialog window will stay open and be visible to the user. We need to close the window with `.destroy()` method.

*Note*: Gtk+ terminology of `close`, `delete` and `destroy` is pretty confusing. I've had to re-read documentation and examples multiple times to double-check my code.

```rust
    dialog.destroy();
```

The dialog window now is not displayed but the corresponding Gtk+ object still exists. Rust's RAII (`Drop`) will clean it up automatically at this point.

```rust
}
```

And now, we just need to run this function. The current version of Gtk-rs has all the necessary pieces to run async GUI code.

The following code will initialize Gtk+ and then run async code to completion and then will quit the program.

```rust
async fn ui_main() {
    alert("Message", "Hello, world").await;    
}

fn main() {
    gtk::init().expect("Failed to initialize Gtk+");

    glib::MainContext::default().spawn_local(async move {
        ui_main().await;
        gtk::main_quit();
    });

    gtk::main();
}
```

*Note*: Some time ago I released a [gtk-future-executor crate](https://crates.io/crates/gtk-future-executor) which could be used to execute the future on Gtk+ event loop.
This crate should be considered obsolete.

The whole example is available [as a gist](https://gist.github.com/dmitryvk/9ce63c7114d9a4eb570b0292ec4ef813).
