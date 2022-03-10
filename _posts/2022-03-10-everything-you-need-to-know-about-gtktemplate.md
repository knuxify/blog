---
layout: post
author: knuxify
tags: gtk pygobject gtk3 gtk4 python gtktemplate
description: The all-in-one guide to working with PyGObject's Gtk.Template.
title: Everything you need to know about Gtk.Template
---

This article aims to cover everything you need to know about PyGObject's `Gtk.Template`, from creating the `.ui` file, through loading it, all the way to working with objects created through the template.

<!--more-->

Some basic knowledge about how GTK and PyGObject work is required; we will not be covering the basics of those two here.

> **Note**: This guide was written with GTK4 in mind; however, everything outlined in this article will work with GTK3 as well.

## `GtkBuilder` vs `Gtk.Template`

There are two ways to work with `.ui` files in PyGObject: with `GtkBuilder` and `Gtk.Template`.

**[`GtkBuilder`](https://docs.gtk.org/gtk4/class.Builder.html)** takes UI definitions from XML files and instantiates them. It is usually used for creating windows or dialogs (although they can be created with `Gtk.Template` as well).

**[`Gtk.Template`](https://pygobject.readthedocs.io/en/latest/guide/gtk_template.html)** is a PyGObject-specific wrapper around GTK's [widget templating functionality](https://docs.gtk.org/gtk4/class.Widget.html#building-composite-widgets-from-template-xml). It can be used to automate the creation of widgets - their children and properties can be defined in a single UI file, rather than having to be created in-code.

The main distinction is that GTK templates are used to **create custom objects** that can be used anywhere just like regular ones, whereas GtkBuilder is used to define entire UIs.

In most cases, you'll want to use `Gtk.Template`.

### UI definitions

Both `GtkBuilder` and `Gtk.Template` use the same [UI definition format](https://docs.gtk.org/gtk4/class.Builder.html#a-gtkbuilder-ui-definition); these definitions are written in XML and describe widgets, their children and their properties.

There is one crucial difference between UI definitions written for `GtkBuilder` and ones written for templates - **templates expect a `<template>` tag** as a direct child of the toplevel `<interface>` tag, whereas standard `GtkBuilder` definitions need it to be an `<object>` tag.

These differences will be further discussed in the next section.

### Benefits of using `Gtk.Template` over writing widget creation code manually

In most cases, you'll have your own widget classes already. If you're creating the widget and its children in-code, it might be worth it to switch to using templates for a couple of reasons:

- It's cleaner;
- It can be inspected in GUI tools like Glade;
- It's easier to port the files to a newer GTK version (for example, if you're porting from GTK3 to GTK4, you can use the `gtk4-builder-tool simplify --3to4` command on your UI file to apply many of the necessary changes automatically).

## Creating the template

As mentioned in the previous section, templates used by `Gtk.Template` are written in XML. They are usually stored in `.ui` files, and contain information about widgets, their children and their properties.

There are a couple of GUI tools that can be used to create UI files, such as [Glade](https://glade.gnome.org/) and the work-in-progress [Cambalache](https://gitlab.gnome.org/jpu/cambalache). However, the general consensus is that [Glade should not be used](https://blogs.gnome.org/christopherdavis/2020/11/19/glade-not-recommended/) due to various compatibility issues.

For this guide, we'll be writing the template file manually - this is also how most developers choose to create UI definition files. It might seem a bit unintuitive at first, but it's actually fairly simple once you get the hang of things.

### A short note about XML syntax

If you've ever written HTML, you'll be somewhat familiar with XML syntax.

A tag is added by adding `<tagname>` and can be closed with `</tagname>`. Alternatively, if the tag will have no children, it can be created using `<tagname/>`, in which case a closing tag will not be necessary.

Comments are enclosed between `<!--` and `-->`.

### Our first template

> **Note**: If you already know how to work with UI files, you can safely skip this section - however, keep the final UI file in mind, as it will be used for all examples in this post.

For our example, we'll create a new widget called **`MyWidget`**, which will inherit from a `GtkBox`. This widget will contain a `GtkImage` with an icon using the `help-about-symbolic` icon name, and a `GtkLabel` saying "Hello World!".

We'll make it do all those things in just a moment. First, let's get familiar with the usual boilerplate you'll find in a template:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<interface>
    <requires lib="gtk" version="4.0"/>
    <template class="MyWidget" parent="GtkBox">
        <!-- ...object properties go here... -->
    </template>
</interface>
```

> **Note**: In some older UI files, you may come across the following:
>
```xml
<!-- interface-requires gtk+ 3.0 -->
```
>
> This is just another way of saying that the interface file requires a specific GTK version, in this case GTK3.

The real magic, however, happens in this part:

```xml
<template class="MyWidget" parent="GtkBox">
    <!-- ...object properties go here... -->
</template>
```

This is what tells our program what the parent class of our widget will be, and all child elements within the `<template>` tags will define the widget's properties and children.

The `class` will have to be set to our widget's GType name (more on that later), and the `parent` - to the class the widget descends from - in our case it's a GtkBox.

#### Adding elements to the template

Now that we have our template set up, let's add our elements to it. To recap - we want to create a widget that has two children: a `GtkImage` and a `GtkLabel`.

Complete with adding properties, doing this *without templates* would look something like this:

```python
box = Gtk.Box(orientation=Gtk.Orientation.VERTICAL)

image = Gtk.Image.new_from_icon_name('help-about-symbolic')
box.append(image)

label = Gtk.Label(label='Hello World!')
box.append(label)
```

Now, compare that to how we'll do it in our template (note that in our case, the `box` becomes our new widget - that is, the object described in the main `<template>` tag):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<interface>
    <requires lib="gtk" version="4.0"/>
    <template class="MyWidget" parent="GtkBox">
        <property name="orientation">vertical</property>
        
        <child>
            <object class="GtkImage">
                <property name="icon-name">help-about-symbolic</property>
            </object>
        </child>
    
        <child>
            <object class="GtkLabel" id="my_label">
                <property name="label">Hello World!</property>
            </object>
        </child>
        
        <child>
            <object class="GtkButton" id="my_button">
                <property name="label">Click me!</property>
                <!-- <signal name="clicked" handler="button_callback"/> -->
            </object>
        </child>
    </template>
</interface>
```

Here are some things that you should take note of:

 - Our template acts just like a regular `GtkBox` would - we add its properties directly to it.
 - Properties are added by adding a `<property>` tag, setting the `name` value to the property's name, and setting the tag's value to the property's value.
 - To add a child, we enclose an `<object>` tag within a `<child>` tag.

These are only the basics of using templates - some object have custom child types, some use custom tags, although the latter is fairly rare. Usually the GTK doc page for the element you want to add will contain information about how to use it in a .ui file.

## Creating the widget class

As mentioned earlier, templates allow us to create our own widgets. To make use of those widgets, though, we need to define a class for them in our code. Let's make one:

```python
from gi.repository import Gtk

@Gtk.Template(filename='/path/to/file.ui')
class MyWidget(Gtk.Box):
    """My new widget."""
    __gtype_name__ = 'MyWidget'

    def __init__+(self):
        """Initializes the widget."""
        super().__init__()
```

Let's analyze this line-by-line.

```python
@Gtk.Template(filename='/path/to/file.ui')
```

This is the `Gtk.Template` decorator, which lets PyGObject know that the class will represent a template, and allows us to link it to said template. The template will then automatically be used to create our widget.

The source for the template can be set in this decorator. There are three options for sourcing the template:

 * from a string, by putting the XML as a string in a variable and providing it to the decorator through the `string` parameter:

```python
my_xml = """\
<?xml version="1.0" encoding="UTF-8"?>
<interface>
  <requires lib="gtk" version="4.0"/>
  <template class="MyWidget" parent="GtkBox">
    <property name="orientation">vertical</property>
    <child>
      <object class="GtkLabel" id="my_label">
         <property name="label">Hello World!</property>
      </object>
    </child>
  </template>
</interface>
"""
  
@Gtk.Template(string=my_xml)
class MyWidget(Gtk.Box):
    __gtype_name__ = 'MyWidget'
```

* from a file, using the `filename` parameter: `@Gtk.Template(filename='/path/to/file.ui')`
* from a GResource, using the `resource_path` parameter: `@Gtk.Template(resource_path='/com/example/myapp/resource.ui')`

Just under the decorator, we define our class:

```python
class MyWidget(Gtk.Box):
    """My new widget."""
    __gtype_name__ = 'MyWidget'
```

This is the actual class definition - the widget is a simple Python object. It's worth noting that **the inherited object must match the parent object**, as set in the template file.

`__gtype_name__` **must be set to the widget's class name**, as seen in the template. The actual Python class name doesn't have to match this - so, for example, you could do this:

```python
class CoolWidget(Gtk.Box):
    """My new widget."""
    __gtype_name__ = 'MyWidget'
```

...and, as long as the `__gtype_name__` matches the class name set in the `<template>` tag, this will refer to `MyWidget`.

## Accessing children

What if we wanted to do something with a widget's child? `Gtk.Template` has a handy tool for that - `Gtk.Template.Child()`.

Let's access our "Hello World" label, for example. In our template, we have set its **ID** to **`my_label`**:

```xml
<object class="GtkLabel" id="my_label">
    <!-- ... -->
</object>
```

To get our child in our code, we can get it using `Gtk.Template.Child()` by declaring it as a variable in the class definition:

```python
class MyWidget:
    """My new widget."""
    __gtype_name__ = 'MyWidget'
    
    my_label = Gtk.Template.Child()
```

The `Gtk.Template.Child()` also takes an argument, `name`, containing the child's ID. If no arguments were provided, `Gtk.Template.Child()` will look for a child with the ID matching the name of the variable we're assigning it to.

Thus, if we want to name our variable differently from our ID, we'll have to specify the ID manually as such:

```python
    cool_label = Gtk.Template.Child('my_label')
```

If your IDs use dashes instead of underscores, you will need to use this method as well:

```python
    my_label = Gtk.Template.Child('my-label')
```

Now, let's change the text of the label. (Of course, if we wanted to change it permanently, we would just change the template - here we change it for demonstration purposes.)

```python
class MyWidget:
    """My new widget."""
    __gtype_name__ = 'MyWidget'
    
    my_label = Gtk.Template.Child()
    
    def __init__(self):
        """Initializes the widget."""
        super().__init__()
        self.my_label.set_label('Good morning!')
```

Now, when we launch our app, the label will say "Good morning!".

## Callback functions

You might have noticed that we have a button in our template, but at the moment it doesn't actually do anything. Let's change that!

To define a callback function, we have to add a function to our widget class, then prepend it with the `@Gtk.Template.Callback()` decorator as such:

```python
class MyWidget:
    """My new widget."""
    __gtype_name__ = 'MyWidget'
    # ...
    @Gtk.Template.Callback()
    def button_callback(self, button, user_data):
        """Callback function that is called when we click the button"""
        print("The button was pressed!")
```

On our template's side, we set the callback function for our button through the `<signal>` tag. The `name` value contains the name of the signal to react to (in this case, `clicked` - you can find the available signals for your widget in [GTK docs](https://docs.gtk.org/gtk4)), and the `handler` value contains the name of the callback function.

```xml
<child>
    <object class="GtkButton" id="my_button">
        <property name="label">Click me!</property>
        <signal name="clicked" handler="button_callback"/>
    </object>
</child>
```

*Note that we previously commented out the `<signal>` tag - adding a handler in the template but not defining the actual callback function causes an error.*

Similarily to `Gtk.Template.Child()`, the `@Gtk.Template.Callback()` decorator takes an optional `name` argument that sets the handler function name, in case your function's name doesn't exactly match the one you provided in the `handler` value in your template.

Now, when we click on the button, the text "The button was pressed!" will appear in the command line.

Let's make our button more useful - we'll combine this with the previous example, and make the button change the label text.

```python
class MyWidget:
    """My new widget."""
    __gtype_name__ = 'MyWidget'
    
    my_label = Gtk.Template.Child()
    
    def __init__(self):
        """Initializes the widget."""
        super().__init__()

    @Gtk.Template.Callback()
    def button_callback(self, button, user_data):
        """Callback function that is called when we click the button"""
        self.my_label.set_label('Good morning!')
```

And just like that, we've made our first example program using PyGObject's `Gtk.Template`!

## Other use cases

In most programs, GTK templates are also used to define windows and dialogs (for a reference/example, GNOME Builder's default Python project template creates an .ui file for the window and about dialog).

## Troubleshooting

### *I tried accessing a child object, but I get errors like `AttributeError: type object 'Child' has no attribute ...`*

You forgot to add the closing brackets (`()`) at the end of `Gtk.Template.Child()`, or you set it inside of your init function - it **must** be defined in the class definition, as seen in the above examples.

### *I tried adding a callback function, but my program says it doesn't exist/throws weird errors and segfaults!*

You most likely forgot to add the `@Gtk.Template.Callback()` decorator, or you accidentally forgot to add the closing brackets at the end of it.

### *My template seems to only partially work/I broke something and the program doesn't launch!*

Check the command line for any errors. Watch out for things like "Unable to retrieve child object ... from class template for type 'MyWidget' while building a 'MyWidget'", template loading errors could halt the loading process and cause errors in functions that are executed.

## See also

* [PyGObject docs on `Gtk.Template`](https://pygobject.readthedocs.io/en/latest/guide/gtk_template.html); check this in case something was added since the creation of this post. Also goes over a few minor things I've ommited here.
* [GTK4 documentation](https://docs.gtk.org/gtk4/index.html)
