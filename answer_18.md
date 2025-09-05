Short version: Camunda Modeler is just a desktop editor that saves a **.bpmn XML file** on disk. Git tracks whatever is on disk. There’s **no special sync** between Modeler and git beyond normal file I/O.

Here’s what that means in practice:

* **Does git detect changes?**
  Yes. As soon as you **save** in Camunda Modeler, the `.bpmn` file’s content changes on disk. `git status` will show it as modified.

* **How “synchronized” are Modeler and git?**
  There’s **no live link**. Modeler doesn’t know about git; it just reads/writes the file. Git sees content changes when they’re saved.

* **If you `git stash`/`git checkout -- <file>`/`git restore` to undo changes…**
  Git will revert the file on disk. If Modeler has that file **open**, it will typically detect the external change and either:

  * auto-reload, or
  * prompt you to reload the file because it changed on disk.
    (If you have unsaved edits in Modeler, you may be prompted to discard or keep them.)

* **If you edit the diagram back to what it was and save—will git say it’s unchanged?**
  Yes, **if the file content is exactly the same bytes** as before. Gotchas that can make it “look changed”:

  * **Whitespace/formatting/attribute order**: Modeler usually preserves IDs and doesn’t churn the XML, but minor reordering can happen.
  * **Line endings** (CRLF vs LF): depends on your git `core.autocrlf` settings.
  * **Trailing newline** differences.

  If the content is identical, `git diff` will be empty and `git status` clean.

* **After a git operation that changes the file while Modeler is open**
  If Modeler doesn’t auto-reload, use **File → Reopen** or close/reopen the tab to ensure you’re seeing the latest from disk.

### Tips / best practices

* Commit `.bpmn` files like any other source file.
* Keep **IDs stable** (Modeler does this by default) to avoid noisy diffs.
* Consider setting a consistent **line ending policy** (`.gitattributes`, `* text=auto`) to reduce platform diffs.
* If diffs get noisy, use `git diff --ignore-all-space` or configure a **.gitattributes** rule for BPMN to ignore whitespace-only changes.

So yes: modify in Modeler → save → git sees change. Use git to revert → Modeler reloads (or prompts). If you revert to the **exact original content**, git shows it as unchanged.
