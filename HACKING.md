Building
========
For build instructions see the README.md

Merge requests
==============
Before filing a pull request run the tests:

```sh
ninja -C _build test
```

Use descriptive commit messages, see

   https://wiki.gnome.org/Git/CommitMessages

and check

   https://wiki.openstack.org/wiki/GitCommitMessages

for good examples. The commits in a merge request should have "recipe"
style history rather than being a work log. See
[here](https://www.bitsnbites.eu/git-history-work-log-vs-recipe/) for
an explanation of the difference. The advantage is that the code stays
bisectably and individual bits can be cherry-picked or reverted.

Checklist
---------
When submitting a merge request consider checking these first. If

- [ ] Does the code use the below coding patterns?
- [ ] Is the commit history in recipe style (see above)?
- [ ] Does the code crash or introduce new CRITICAL or WARNING
      messages in the log or when run form the console. If so, fix
      these first.

If any of the above criteria aren't met yet it's still fine to open a
merge request marked as draft. Please indicate why you consider it
draft in this case.

Coding Patterns
===============

Coding Style
------------
We're mostly using [libhandy's Coding Style][1].

These are the differences:

- We're not picky about GTK+ style function argument indentation, that is
  having multiple arguments on one line is also o.k.
- For callbacks we additionally allow for the `on_<action>` pattern e.g.
  `on_nm_signal_strength_changed()` since this helps to keep the namespace
  clean.
- Since we're not a library we usually use `G_DEFINE_TYPE` instead of
  `G_DEFINE_TYPE_WITH_PRIVATE` (except when we need a deriveable
  type) since it makes the rest of the code more compact.

Source file layout
------------------
We use one file per GObject. It should be named like the GObject without
the phosh prefix, lowercase and '_' replaced by '-'. So a hypothetical
`PhoshThing` would go to `src/thing.c`. If there are likely name
clashes add the `phosh-` prefix (see e.g. `phosh-wayland.c`). The
individual C files should be structured as (top to bottom of file):

  - License boilerplate
    ```c
    /*
     * Copyright (C) year copyright holder
     *
     * SPDX-License-Identifier: GPL-3-or-later
     * Author: you <youremail@example.com>
     */
    ```
  - A log domain
    ```C
    #define G_LOG_DOMAIN "phosh-thing"
    ```
    Usually the filename with `phosh-` prefix.
  - `#include`s:
    Phosh ones go first, then glib/gtk, then generic C headers. These blocks
    are separated by newline and each sorted alphabetically:

    ```
    #define G_LOG_DOMAIN "phosh-settings"

    #include "settings.h"
    #include "shell.h"

    #include <gio/gdesktopappinfo.h>
    #include <glib/glib.h>

    #include <math.h>
    ```

    This helps to detect missing headers in includes.
  - docstring
  - property enum
    ```c
    enum {
      PROP_0,
      PROP_FOO,
      PROP_BAR,,
      LAST_PROP
    };
    static GParamSpec *props[LAST_PROP];
    ```
  - signal enum
    ```c
    enum {
      FOO_HAPPENED,
      BAR_TRIGGERED,
      N_SIGNALS
    };
    static guint signals[N_SIGNALS] = { 0 };
    ```
  - type definitions
    ```c
    typedef struct _PhoshThing {
      GObject parent;

      ...
    } PhoshThing;

    G_DEFINE_TYPE (PhoshThing, phosh_thing, G_TYPE_OBJECT)
    ```
  - private methods and callbacks (these can also go at convenient
    places above `phosh_thing_constructed ()`
  - `phosh_thing_set_property ()`
  - `phosh_thing_get_property ()`
  - `phosh_thing_constructed ()`
  - `phosh_thing_dispose ()`
  - `phosh_thing_finalize ()`
  - `phosh_thing_class_init ()`
  - `phosh_thing_init ()`
  - `phosh_thing_new ()`
  - Public methods, all starting with the object name(i.e. `phosh_thing_`)

  The reason public methods go at the bottom is that they have declarations in
  the header file and can thus be referenced from anywhere else in the source
  file.

CSS Theming
-----------
For custom widget always set the css name using `gtk_widget_class_set_css_name ()`.
There's no need set an (additional) style class in the ui file.

*Good*:

```c
static void
phosh_lockscreen_class_init (PhoshLockscreenClass *klass)
{
  …
  gtk_widget_class_set_css_name (widget_class, "phosh-lockscreen");
  …
}
```

*Bad*:

```xml
  <template class="PhoshLockscreen" parent="…">
  …
     <style>
         <class name="phosh-lockscreen"/>
      </style>
  …
  </template>
```

Properties
----------

### Signal emission on changed properties
Except for `G_CONSTRUCT_ONLY` properties use `G_PARAM_EXPLICIT_NOTIFY` and notify
about property changes only when the underlying variable changes value:


```
static void
on_present_changed (PhoshDockedInfo *self)
{
  …

  if (self->enabled == enabled
    return;

  self->enabled = enabled;
  g_object_notify_by_pspec (G_OBJECT (self), props[PROP_ENABLED]);
}

static void
phosh_docked_info_class_init (PhoshDockedInfoClass *klass)
{
  …

  /**
   * PhoshDockedInfo:enabled:
   *
   * Whether docked mode is enabled
   */
  props[PROP_PRESENT] =
    g_param_spec_boolean ("present", "", "",
                          G_PARAM_READABLE |
                          G_PARAM_STATIC_STRINGS |
                          G_PARAM_EXPLICIT_NOTIFY);

  …
}
```

This makes sure we minimize notificatons on changed property values.

### Property documentation
Prefer a docstring of filling in the properties `nick` and `blurb`. See
example above.

API contracts
-------------
Public (non static) functions must check the input arguments at the
top of the function. This makes it easy to reuse them in other parts
and makes API misuse easy to debug via `G_DEBUG=fatal-criticals`. You
usually want to check argument types and if the arguments fulfill the
requirements (e.g. if they need to be non-NULL).

*Good*:

```c
void
phosh_foo_set_name (PhoshFoo *self, const char *name)
{
  GtkWidget *somewidget;

  g_return_if_fail (PHOSH_IS_FOO (self));
  g_return_if_fail (!name);

  …
}
```

[1]: https://gitlab.gnome.org/GNOME/libhandy/blob/master/HACKING.md#coding-style
