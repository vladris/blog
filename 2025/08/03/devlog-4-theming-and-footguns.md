# DevLog 4: Theming and Footguns

This is going to be a fun post! First, I want to show how I implemented theming
for Flow. With its focus on providing an inspiring environment, theming was
min-bar for the app. Fortunately, implementing theming is fairly
straight-foward. Until it isn't. In the first part I'll go over how I gave the
Electron app its unique look and feel. In the second part, I'll cover the
strange snag I hit.

## Theming

Making something themeable in the web world is easy with CSS variables:

```css
:root {
    --color-bg-primary: #fff;
    --color-bg-secondary: #fff;
    --color-bg-tertiary: #fff;

    --color-fg-primary: #000;
    --color-fg-secondary: #000;
    --color-fg-accent: #000;

    /* ... */
}

.latte-theme {
    --color-bg-primary: #f9f5f0;
    --color-bg-secondary: #e6dfd3;
    --color-bg-tertiary: #f0ebe0;

    --color-fg-primary: #5a4534;
    --color-fg-secondary: #7a6b4f;
    --color-fg-accent: #c74343;

    /* ... */
}
```

The `:root` style will provide the defaults while subsequent classes can
override it. In the above example, `.latte-theme` is a fragment of the app's
Latte theme.

On first boot, before the user can configure the theme, I want the app to
respect the operating system's dark mode. We can do this with the
`prefers-color-scheme: dark` media query:

```css
:root {
    --color-bg-primary: #fff;
    --color-bg-secondary: #fff;
    --color-bg-tertiary: #fff;

    --color-fg-primary: #000;
    --color-fg-secondary: #000;
    --color-fg-accent: #000;

    /* ... */

    @media (prefers-color-scheme: dark) {
        --color-bg-primary: #000;
        --color-bg-secondary: #000;
        --color-bg-tertiary: #000;

        --color-fg-primary: #fff;
        --color-fg-secondary: #fff;
        --color-fg-accent: #fff;

        /* ... */
    }
}
```

With this, on first boot the app should use the "dark mode" colors if the OS is set to dark mode.

Now all this happens on the render thread. Electron boots up fairly fast, but
users might still see a blank window for an instant between the time the main
process has started but the render processes hasn't finished booting. For that
case, I wanted the blank window to still respect the OS setting, otherwise when
in dark mode, users might see a flashing white window. That's because Electron's
default background color is white.

The following code sets up the main window:

```typescript
const mainWindow = new BrowserWindow({
  width: 1200,
  height: 800,
  minWidth: 300,
  minHeight: 400,
  titleBarStyle: "hiddenInset",
  backgroundColor: nativeTheme.shouldUseDarkColors ? "#000" : "#fff",
  /* ... */
});
```

The relevant part is the `backgroundColor`. The
`nativeTheme.shouldUseDarkColors` API is the way to check whether the OS is in
dark more inside the main process (as opposed to the render process where we can
use media queries).

### Settings

The app's theme is one of its many settings. The way settings work in general is
also an interesting topic. I've been using `electron-store` to store the app
settings. This package abstracts a native store which supports typed schemas and
version migrations of settings:
<https://github.com/sindresorhus/electron-store>.

The store works on the main process, which means I had to implement IPC to
get/set settings on the render process. This type of IPC is common for Electron
apps and follows the same pattern as I describe in the [Commanding
post](https://vladris.com/blog/2025/05/08/devlog-3-commanding.html). As a quick
recap, the render process talks to the main process via
`window.electronAPI.invoke` and the main process receives and handles these
events. Here's how the render process asks for a setting item:

```typescript
function getStoreItem<K extends keyof StoreType>(
    key: K
): Promise<StoreType[K] | undefined> {
    return window.electronAPI.invoke(GET_STORE_ITEM, key);
}
```

Here `StoreType` is the strongly typed store schema and `GET_STORE_ITEM` is just
a string to identify the event type.

On the main process side, this is handled via:

```typescript
ipcMain.handle(GET_STORE_ITEM, (_, key: keyof StoreType) => {
    const value = store.get(key);
    return value;
});
```

With IPC taken care of, I implemented a React provider that wraps the IPC code
and exposes the settings via a `SettingsContext`. Theming becomes super-easy
with this. For example, inside the React component implementing the main view:

```typescript
const EditorContent: React.FC = () => {
    const { theme, font, fontSize, autoIndent, lineHeight, paragraphSpacing } = 
		useSettings();

    /* ... */

    return (
        <div className={`app-container ${theme}`}>
            <TitleBar />
            <SideBar />

            <Editor {...editorProps} />
            <StatusBar />
        </div>);
    );
}        
```

Note any component can easily get the settings from the provider and, in the
case of theming, we simply use it as a `className` prop inside the JSX.

The provider also allows consumers to update the various settings. For example,
for theming, it exposes an `updateTheme()` API that allows components to update
the setting.

### Multiple windows

When working with theming though, a reasonable expectation would be to have the
theme change across the all open app windows. So as soon as I select the `Latte`
theme from the theme picker in the Settings window, the main window should
change colors.

The existing IPC mechanism I described so far isn't sufficient for that: the
render process can get and set settings, but a key missing piece is that, as
soon as a setting changes, the main process needs to broadcast this to all
window instances.

Previously I shared the implementation of the main process's `GET_ITEM` handler:

```typescript
ipcMain.handle(GET_STORE_ITEM, (_, key: keyof StoreType) => {
    const value = store.get(key);
    return value;
});
```

The `SET_ITEM` handler does a bit more work:

```typescript
ipcMain.handle(
    SET_STORE_ITEM,
    <K extends keyof StoreType>(
    _: unknown,
    key: K,
    value?: StoreType[K]
) => {
    store.set(key, value);

    BrowserWindow.getAllWindows().forEach((win) => {
        win.webContents.send(SETTING_CHANGED, { key, value });
    });
}
```

The function takes a key of type `K` which needs to be part of the store schema
and an optional value of that type. This is the standard to call the `store`
API, which happens on the first lie (`store.set(key, value);`). Then, once the
value is stored, I use the Electron API to sent a `SETTING_CHANGED` event to all
windows and pass them the `key` that changed and the new `value`.

On the render process side, we listen to the event on the window, and update the
corresponding setting in the provider:

```typescript
window.electronAPI.on(SETTING_CHANGED, data => {
    const { key, value } = data;
    /* Update setting */
}
```

This way, changes reflect instantly across the app.

### Additional considerations

A benefit of using CSS variables for themes is that it's super easy to implement
the different theme cards in the Settings window. A theme card is a React
component to which we apply the theme we want it to display, and since we have a
CSS class for each theme, we can put together the gallery with very little code:

```typescript
const ThemeCard: React.FC<ThemeCardProps> = ({ title, className, current, onClick }) => {
    return (
        <div className={`theme-card ${className}`} onClick={onClick} data-current={current}>
            <div className="theme-card-content">
                <h2 className="theme-card-title">{title}</h2>
                <p>"The universe is made of stories, not atoms."</p>
                {/* Rest of card content */}
            </div>
        </div>
    );
};
```

The gallery becomes just a sequence of `ThemeCard`s with different props.

## On footguns

This brings me to the funny part of the journey. I started building Flow as a
macOS app but based it on Electron on the idea that it will be easy to bring it
to Windows. As I started looking into a Windows version, I hit a strange snag I
didn't consider when I was focused on macOS: the menu.

On macOS, the menu bar shows at the top of the screen. I described in the
[Commanding post](https://vladris.com/blog/2025/05/08/devlog-3-commanding.html)
how the commands end up in a registry which I serialize and send to the main
process to set up the native menu.

On Windows on the other hand, the menu bar appears on the window itself. And, of
course, the native menu is not themeable. Worse, because I'm theming the whole
app, I use a custom title bar since it would otherwise get the default system
colors. The menu bar is part of the browser window on Windows, so there is no
way to put anything before it.

The only reasonable conclusion, which I guess all Electron apps that support
theming do, is that I need a custom menu bar implementation for Windows. That
is, a full React menu system that binds to the commanding infrastructure.

The reason I find this extremely funny is that I picked Electron to write once
and run everywhere, and it seems I'll end up with a ton of platform-specific
code: on macOS, set up the native menu on the main process; on Windows, set up a
custom menu on the render process.

## Summary

In this post I talked about how I implemented theming in Flow, and settings more
generally:

* Themes are implemented as CSS classes overriding CSS variables.
* Some boot considerations: being aware of dark mode before user picked a
  specific theme.
* I use `electron-store` to store the settings.
* A `SettingsProvider` on the render process handles IPC.
* The main process broadcasts all setting changes to all active windows so any
  change is reflected instantly across the whole app.
* The interesting issue I ran into where on Mac the menu is handled by the main
* process but on Windows, as long as I want custom theming throughout the app,
  the menu must be custom-built thus run in the render process.

*This post was written with Flow. Try it out [here](https://saturn9.studio/flow/).*
