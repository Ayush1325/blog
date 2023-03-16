+++
title = "Using KConfig with Rust"
description = "A guide on how to use KConfig KDE Framework from Rust"
date = "2022-03-14T22:19:49+05:30"

[taxonomies]
categories = ["post"]
tags = ["rust", "kde", "sok22"]
+++

Hello everyone, I am currently working on KConfig bindings for Rust as a part of the Season of KDE 2022. The wrappers for most of the significant aspects of KConfig are complete, so I decided to rewrite the [Introduction to KConfig Docs](https://develop.kde.org/docs/use/configuration/introduction/) in Rust. The bindings are still not stable and will probably change before the end of the Season of KDE. Still, this post should also help me test out the bindings outside tests. The kconfig bindings can be found [here](https://invent.kde.org/oreki/kconfig-rs).

<!-- more -->

The bindings currently use the git version of [qttypes](https://github.com/woboq/qmetaobject-rs) since I had to merge some upstream changes that are needed for these bindings. So they are not ready for prime time just yet.

# Introduction to KConfig
## The KConfig Class

The KConfig object is used to access a given configuration object. The config object can be created in the following ways:

```rust
// a plain old read/write config object
let config = KConfig::with_file("myapprc");

// a specific file in the filesystem
// currently must be an INI style file
let full_path = KConfig::with_file("/etc/kderc");

// not merged with global values
let global_free = KConfig::new("config", OpenFlags::NO_GLOBALS, QStandardPathLocation::AppDataLocation);

// not merged with globals or the $KDEDIRS hierarchy
let simple_config = KConfig::new("simple_rc", OpenFlags::SIMPLE_CONFIG, QStandardPathLocation::AppDataLocation);

// outside the standard config resource
let data_resource = KConfig::new("data", OpenFlags::SIMPLE_CONFIG, QStandardPathLocation::AppDataLocation);

// with custom backend
let custom_backend = KConfig::with_backend("config", "backend", QStandardPathLocation::AppDataLocation);
```

## Special Configuration Objects
Each application has its own configuration object that uses the name provided to KAboutData appended with “rc” as its name. So an app named “myapp” would have the default configuration object of “myapprc” (located in `$HOME/.config/`). This configuration file can be retrieved in this way:
```rust
let config  = KSharedConfigPtr::default();
```
The default configuration object for the application is accessed when no name is specified when creating a KConfig object. So we could also do this instead - but it would be slower because it would have to parse the whole file again:
```rust
let config = KConfig::default();
```
Finally there is a global configuration object, `kdeglobals`, that is shared by every application. It holds such information as the default application shortcuts for various actions. It is “blended” into the configuration object if the `kconfig::kconfig::OpenFlags::INCLUDE_GLOBALS` flag is passed to the KConfig constructor, which is the default.

## Commonly Useful Methods
To save the current state of the configuration object we call the `sync()` method. This method is also called when the object is destroyed. If no changes have been made or the resource reports itself as non-writable (such as in the case of the user not having write permissions to the file) then no disk activity occurs. `sync()` merges changes performed concurrently by other processes - local changes have priority, though.

If we want to make sure that we have the latest values from disk we can call `reparse_configuration()` which calls `sync()` and then reloads the data from disk.

If we need to prevent the config object from saving already made modifications to disk we need to call `mark_as_clean()`. A particular use case for this is rolling back the configuration to the on-disk state by calling `mark_as_clean()` followed by `reparse_configuration()`.
Listing all groups in a configuration object is as simple as calling `group_list()` as in this code snippet:
```rust
let config = KSharedConfigPtr::default();

let group_list = config.group_list();
for group in &group_list {
	log::debug!("next group: {}", group);
}
```

## KSharedConfig
The KSharedConfig class is a reference counted pointer to a KConfig . It thus provides a way to reference the same configuration object from multiple places in your application without the extra overhead of separate objects or concerns about synchronizing writes to disk even if the configuration object is updated from multiple code paths.
Accessing a KSharedConfig object is as easy as this:
```rust
let config = KSharedConfig::open_config("ksomefilerc", OpenFlags::FULL_CONFIG, QStandardPathLocation::GenericConfigLocation);
```
KSharedConfig is generally recommended over using KConfig itself.

## KConfigGroup
Now that we have a configuration object, the next step is to actually use it. The first thing we must do is to define which group of key/value pairs we wish to access in the object. We do this by creating a KConfigGroup object:
```rust
let mut config = KConfig::default();
let mut general_group = config.group("General");
let colors_group = config.group("Colors");
```
Config groups can be nested as well:
```rust
let sub_group2 = general_group.group("Dialogs");
```

## Reading Entries
With a KConfigGroup object in hand reading entries is now quite straight forward:
```rust
let account_name = general_group.read_qstring_entry("Account").unwrap();
let color = QColor::from_qvariant(colors_group.read_qvariant_entry("background", QColor::from(Qt::white).to_qvariant()).unwrap());
let list = QStringList::from_qvariant(general_group.read_qvariant_entry("List", QStringList::default().to_qvariant()).unwrap());
let path = general_group.read_path_entry("SaveTo", "defaultPath".into());
```
In the example above, one can mix reads from different KConfigGroup objects created on the same KConfig object. The bindings currently contain three read methods:
1. `read_qstring_entry()` for reading `QString` values
2. `read_entry()` for reading `QVariant` values that can later be converted to any of the other types
3. `read_path_entry()` which returns a file system path.

If no such key currently exists in the configuration object, the default value is returned instead. If there is a localized (e.g. translated into another language) entry for the key that matches the current locale, that is returned.

## Writing Entries
Setting new values is similarly straightforward:
```rust
general_group.write_qstring_entry("Account", "accountName".into(), WriteConfigFlags::NORMAL);
general_group.write_path_entry("SaveTo", "savePath".into(), WriteConfigFlags::NORMAL);
color_group.write_qvariant_entry("background", QColor::from_name("white").into(), WriteConfigFlags::NORMAL);
config.sync();
```
Once we are done writing entries, `sync()` must be called on the config object for it to be saved to disk. We can also simply wait for the object to be destroyed, which triggers an automatic `sync()` if necessary.

## KDesktopFile: A Special Case
When is a configuration file not a configuration file? When it is a desktop file. These files, which are essentially configuration files at their heart, are used to describe entries for application menus, mimetypes, plugins and various services.
When accessing a .desktop file, one should instead use the KDesktopFile class which, while a KConfig class offering all the capabilities described above, offers a set of methods designed to make accessing standard attributes of these files consistent and reliable.

## Kiosk: Lockdown and User/Group Profiles
KConfig provides a powerful set of lockdown and configuration definition capabilities, collectively known as “Kiosk”, that many system administrators and system integrators rely on. While most of this framework is provided transparently to the application, there is occasion when an application will want to check on the read/write status of a configuration object.
Entries in configuration objects that are locked down using the kiosk facilities are said to be immutable. An application can check for immutability of entire configuration objects, groups or keys as shown in this example:
```rust
let config = KGlobal::config();

if config.is_immutable() {
    log::debug!("configuration object is immutable");
}

let group = config.group("General".into());
if group.is_immutable() {
    log::debug!("group General is immutable");
}

if group.is_entry_immutable("URL".into()) {
    log::debug!("URL entry in group General is immutable");
}
```
This can be useful in particular situations where an action should be taken when an item is immutable. For instance, the KDE panels will not offer configuration options to the user or allow them to otherwise change the order of applets and icons when the panel’s configuration object is marked as immutable.

# Conclusion
Since most of the work on base KConfig bindings is done, I will now be working on KConfigXT implementation in Rust. Instead to a wrapper, I will be generating Rust code from the kcfg files. Some work has already been done, however, it does not seem like I will be able to complete it anytime soon. Anyone interested in contributing to Rust/QML development is welcome. Here are some important project links:
1. [KConfig Rust Bindings](https://invent.kde.org/oreki/kconfig-rs): The bindings I am currently working on.
2. [qmetaobject](https://github.com/woboq/qmetaobject-rs)
3. [cxx-qt](https://github.com/KDAB/cxx-qt/)
