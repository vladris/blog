# DevLog 3: Commanding

This article is going to be the part 2 of the *formatting and commanding* I
built for [Flow](https://saturn9.studio/flow/). In the [previous
post](https://vladris.com/blog/2025/04/04/devlog-2-formatting.html) I covered
formatting and the various corner cases we need to keep in mind when
implementing something as simple as *toggle bold on the current selection*. Last
post dealt with what we need to do to update the document when such a command
comes in. Now that that’s taken care of, let’s see how we can expose of these
formatting commands (and more) to an app hosting our editor.

Let’s go over what a well-behaved editor needs to do. Commands like “toggle
bold” can be shown to the user in multiple places: app menu bar, toolbar,
context menu etc. We need to make sure invoking a command for any of these
places does the same thing. More so, a command like “toggle bold” is stateful.
If the selected text is already bold, we should show a checkmark next to the
Bold menu item and show the Bold button in the toolbar depressed. If the editor
is, for whatever reason, in read-only mode, the command should be disabled and
the corresponding UI element should be grayed out.

Of course, we need to be consistent - if a command is disabled, it should be
disabled everywhere, not show up as enabled in the menu bar but disabled in the
toolbar or vice-versa. Also, invoking it from anywhere should yield the exact
same results.

Even before diving into the technical details, given the requirements it’s quite
obvious we need some form of object representation for a command, including some
state, like whether the command is active (e.g. current selection is Bold) and
enabled, and an action we can invoke when we want to apply the command.

This will be the focus of this post, with some extra complications: while
formatting is neatly contained within the editor, commands allow the hosting app
to change editor state. More so, I built Flow as an Electron app. An Electron
app runs a *main process*, running the Node code and communicating with the OS,
and a *render process*, running a browser. The editor is, of course, hosted on
the render process, but the menu bar and context menu are handled by the main
process as they are relying on OS-native features. The commanding infrastructure
needs to keep command state in sync across processes!

Another complication is that we might have commands that aren’t related to
formatting. In fact, they might not involve the editor at all. A few examples:
in Flow, you can switch from Focus Mode to Edit Mode to Read Mode to Source
Mode. These reconfigure the view but don’t mutate the document. We also have
commands like *show the settings window* which has nothing to do with the
editor. And while these could, of course, be implemented through a completely
different architecture, it makes sense to weave them into the general commanding
infrastructure. 

Here’s what I did for Flow:

## Commands

The basic building block is the `Command`:

```typescript
interface Command {
    id: string;
    label: string;
    accelerator?: string;
    enabled: () => boolean;
    active?: () => boolean;
    execute: (...args: any[]) => unknown;
}
```

In Flow, a `Command` has:

* A unique id.
* A label (what’s going to be shown in the menu for example).
* An optional keyboard shortcut (`accelerator`).
* A function that should tell us whether the command is enabled or not.
* An optional function that should tell us whether we should show a checkmark
  next to its menu entry (or show its button depressed) - note this is optional
  as not all commands are “toggle” commands.
* A way to invoke the command passing some arguments to it.

This is pretty straight-forward, but things get a bit more complicated: since we
need to show command state across the main and render processes, we need a way
to pass command data between them. Functions are not serializable, so we also
need a `CommandData` way to pass information:

```typescript
interface CommandData {
    id: string;
    label: string;
    accelerator?: string;
    enabled: boolean;
    active?: boolean;
    type?: "normal" | "checkbox";
}
```

Thinks of `CommandData` as an instance in time of a `Command`. It captures a
command’s state and can pass it from the render process where the editor lives
to the app process. You might have noticed an extra property: `type` tells the
UI whether this command should show up as a “normal” command, or as a command
that can have a checkmark next to it. A “normal” command without a checkbox is
something like *insert a horizontal rule*. A command with a checkbox is
something like *toggle bold*.

We also have a straight-forward way of serializing a `Command` into its
`CommandData`:

```typescript
function commandDataFromCommand(command: Command): CommandData {
    return {
        id: command.id,
        label: command.label,
        accelerator: command.accelerator,
        enabled: command.enabled(),
        active: command.active ? command.active() : undefined,
        type: command.active ? "checkbox" : "normal",
    };
}
```

This way, we can pass the current command state from the render process to the
main process whenever the editor state changes in any way.

## The command registry

Of course, we will have many commands so need a way to keep them together. Enter
the `CommandRegistry`:

```typescript
class CommandRegistry {
    private commands = new Map<string, Command>();
    private listeners = new Set<() => void>();

    register(command: Command) {
        this.commands.set(command.id, command);
    }

    getCommand(id: string): Command | undefined {
        return this.commands.get(id);
    }

    getAllCommands(): Command[] {
        return Array.from(this.commands.values());
    }

    getCommandData(): SerializedCommandRegistry {
        const commands = new Map<string, CommandData>();
        this.commands.forEach((command) => {
            commands.set(command.id, commandDataFromCommand(command));
        });
        return commands;
    }

    onUpdate(listener: () => void) {
        this.listeners.add(listener);
    }

    offUpdate(listener: () => void) {
        this.listeners.delete(listener);
    }

    emitUpdate() {
        this.listeners.forEach((listener) => listener());
    }
}
```

The `CommandRegistry` wraps a set of commands and allows consumers to subscribe
to notifications for when the state changes.

The `register()`, `getCommand()`,  `getAllCommands()` are trivial.
`getCommandData()` serializes the whole registry. It’s return type,
`SerializedCommandRegistry`, is:

```typescript
type SerializedCommandRegistry = Map<string, CommandData>;
```

The remaining functions, `onUpdate()`, `offUpdate()`, and `emitUpdate()` handle
notifications. Subscribers can start listening to command state change updates
via `onUpdate()` and unsubscribe via `offUpdate()`. Finally, `emitUpdate()` is
called by whoever changes command state such that subscribers are notified of
the change.

Let’s look at a concrete example. Here is how the *toggle bold* command is
implemented:

```typescript
const boldCommand: Command = {
    id: "bold",
    label: "Bold",
    accelerator: "CmdOrCtrl+B",
    enabled: () => view.editable,
    active: () => isRangeAllBold(view.state),
    execute: () => toggleBold(view.state, view.dispatch),
};
```

We covered `isRangeAllBold()` and `toggleBold()` in the [previous
post](https://vladris.com/blog/2025/04/04/devlog-2-formatting.html). Note that
they both need references to the ProseMirror document state and `toggleBold()`
also needs a way to dispatch transactions, so this `Command` is defined inside
the editor code.

An editor instance will have an associated command registry. Flow initializes
the editor with a `boot()` function:

```typescript
function boot(element: HTMLElement, options?: EditorOptions) {
    let state = EditorState.create({
        /* ... */
    });

    const view = new EditorView(element, {
        state,
        attributes: {
            /* ... */
        },
    });

    const commandRegistry = new CommandRegistry();
    initializeEditorCommands(view, commandRegistry);
    view.commandRegistry = commandRegistry;

    /* ... */

    const editorContext = {
        view,
        commandRegistry,

        /* ... */
    };

    return editorContext;
}
```

When we boot an editor instance, we first create a `state` and a `view` - this
is standard in ProseMirror. Next, we instantiate a `CommandRegistry`, initialize
it, and tack it on to the view. This snipped omits some extra stuff that happens
during boot as it’s not the focus of this post. We create an `editorContext`
which we return to whoever initialized our editor, including the `view` and the
`commandRegistry`.

The `initializeEditorCommands()` function is straight-forward: it consists of a
bunch of command definitions like the `boldCommand` we saw above and calls to
`commandRegistry.register()`. This popoulates the registry with all the editor
commands.

## Inter-process communication

We also need to ensure `emitUpdate()` is called whenever the editor state
changes. For example, if the selection moves from normal text to bolded text,
the `boldCommand`'s `active()` will return `true`, so the menu bar entry for
`Bold` should show a checkmark next to it. But because the MacOS menu lives on
the main process, we can't evaluate `active()` there, rather we need to
serialize all commands into `CommandData` and pass them from the render process
to the main process. We evaulate `active()` during serialization. `emitUpdate()`
lets the app know state changed so it should update the main process.

To ensure we always call `emitUpdate()` on editor state changes, we can replace
the ProseMirror view's `dispatch` with an updated version:

```typescript
const originalDispatch = view.dispatch.bind(view);
view.dispatch = (tr: Transaction) => {
    originalDispatch(tr);
    commandRegistry.emitUpdate();
};
```

The view's `dispatch()` function is how all transactions are applied to the
document state, so this is a good choke point to catch all state changes.

Then we can use the Electron API to send the serialized menu to the main
process:

```typescript
commandRegistry.onUpdate(() => {
    window.electronAPI.send(UPDATE_MENU, commandRegistry.getCommandData());
});
```

`window.electronAPI` is a standard way to set up inter-process communication
in an Electron app. the `electronAPI` object is made available on the browser
window. It looks like this:

```typescript
declare global {
    interface Window {
        electronAPI: {
            send: (channel: string, data: any) => void;
            on: (channel: string, listener: (...args: any[]) => void) => void;
            /* ... */
        }
    }
}
```

`send()` passes data to the main process while `on()` is called when the main
process wants to talk to the render process. In our case, we call `send()` with
`UPDATE_MENU` and the serialized command registry. `UPDATE_MENU` is just a
constant agreed upon by both processes.

End-to-end, the system works like this:

* The user moves the cursor on bold text.
* Moving the cursor is a transaction in ProseMirror, so the view's `dispatch()`
  function ends up being invoked. Our updated version of `dispatch()` calls
  `commandRegistry.emitUpdate()` internally.
* The event handler will get called, and it will `send()` an `UPDATE_MENU` to
  the main process, passing it a serialized command registry.
* When serializing the command registry, `active()` is evaluated for each
  command and serialized to a boolean. For `boldCommand`, this will now change
  from `false` to `true`.
* The menu bar will be updated on the main process to show a checkmark next to
  the `Bold` menu item.

And the other way around, on the render process we listen for `EXECUTE_COMMAND`:

```typescript
window.electronAPI.on(EXECUTE_COMMAND, (commandId: string) => {
    commandRegistry.getCommand(commandId)!.execute();
});
```

When the user selects `Bold` from the menu bar, the main process will send an
`EXECUTE_COMMAND` to the render thread, with the `boldCommand` as `commandId`.
The render process will call the command's `execute()` function, which, in this
case, will call `toggleBold()`.

As of now, we serialize and transfer the whole command registry on each state
change. There are probably more efficient ways of doing this, but the current
implementation is simple and the frequency of updates is low enough (on editor
state changes, meaning on user input) that I'm not concerned about the
performance impact of this approach.

## Additional commands

The editor's host can add its own commands to this registry. For example, Flow
hooks up the mode change commands. Here's `focusMode`:

```typescript
commandRegistry.register({
    id: "focusMode",
    label: "Focus Mode",
    enabled: () => true,
    active: () => modeRef.current === "FocusMode",
    execute: () => {
        setMode("FocusMode");
    },
});
```

Where `modeRef` here is a React ref to the curent mode. 

The distinction here is that the editor commands (like `boldCommand`) are
registered by the editor itself on boot via `initializeEditorCommands()`. Then
the host adds more commands to the registry, like `foucsMode`, which don't
directly impact the editor state.

## Commanding surfaces

We have a `commandRegistry` which we keep up to date with the editor's state
and synchronized with the main process. Why do we need a centralized registry?
Because the same commands can appear in different places. So far we only talked
about the app menu bar, but that is not the only commanding surface.

Flow also has a context menu. Right-clicking inside the editor will pop over
a context menu with a subset of the commands. `Bold` is one of them too. One
advantage of the centralized `commandRegistry` is that both the app menu bar
and the context menu get their data from the same source. So if `Bold` should
have a checkmark next to it (selection is on bold text), then the checkmark
will show up consistently across the app menu bar and the context menu.

Not yet implemented in Flow, but coming soon: a toolbar. A toolbar would also
source it's state from the `commandRegistry`. The same way we determine whether
to show a checkmark next to the `Bold` menu entry based on the `active`
property, we would determine whether to show the `Bold` toolbar button
depressed.

The shared `commandRegistry` will ensure the commands look and behave the same
regardless of where they are surfaced.

## Summary

In this post I covered commanding and how Flow implements it.

* `Command` objects represent commands.
* The `CommandRegistry` collects these into a catalog that can be serialized (to
  be sent between processes) and that can notify listeners of changes.
* We automated emitting updates by modifying the ProseMirror view's
  `dispatch()` method.
* We looked at how Electron handles inter-process communication (via `send()`
  and `on()`) and how we're using these with the commands.
* We talked about commanding surfaces and how the common registry can be used
  across different UI controls like app menu bar, context menu, and toolbars.

This was the second part of the *formatting and commanding*. In the [previous
post](https://vladris.com/blog/2025/04/04/devlog-2-formatting.html) I covered
how formatting is implemented inside the editor, while in this post we went over
the layer on top of that - how formatting actions are represented as commands,
kept in sync with the editor state, and surfaced in the UI.

Flow is now available on the [Mac App
Store](https://apps.apple.com/us/app/flow-by-saturn9/id6744873955?mt=12).
