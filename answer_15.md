Great 👍 let’s walk step by step how you’d make the change visually in **Camunda Modeler**. No XML editing needed.

---

## 🔹 Step-by-Step in Camunda Modeler

1. **Open your process**

   * Start Camunda Modeler.
   * Open the existing `.bpmn` file from your project.

2. **Locate the current flow**

   * You should see something like:

     ```
     DoSomething → DoAnotherThing → DoLastThing → SomeWorker
     ```
   * Confirm that `DoLastThing` has an outgoing arrow to `SomeWorker`.

3. **Insert an Exclusive Gateway**

   * From the left toolbar (shapes panel), drag the **diamond icon** (Exclusive Gateway) onto the canvas.
   * Drop it **after “DoAnotherThing”** (between “DoAnotherThing” and “DoLastThing”).

4. **Reconnect the flows**

   * Delete the arrow that goes directly from `DoAnotherThing` → `DoLastThing`.
   * Connect `DoAnotherThing` → **Gateway**.
   * From the gateway, draw one arrow to **SomeWorker**.
   * Draw another arrow to **DoLastThing**.

5. **Add conditions to flows**

   * Click the arrow going to **SomeWorker**.
   * In the right-hand properties panel, find **Condition Expression**.
     Enter:

     ```
     ${goToSomeWorker}
     ```
   * Click the arrow going to **DoLastThing**.
     Enter:

     ```
     ${!goToSomeWorker}
     ```

6. **Save**

   * File → Save (this updates the `.bpmn` XML).
   * Commit to Git if you’re using version control.

---

## 🔹 What you’ll see visually

* Before:
  A straight line: `DoAnotherThing → DoLastThing → SomeWorker`.

* After:

  ```
  DoAnotherThing → ◇ (Gateway)
                     ↘ yes → SomeWorker
                     ↙ no  → DoLastThing
  ```

---

## 🔹 Verify

* Deploy your process (or restart your app so it auto-deploys).
* Start a new instance in Operate.
* If `goToSomeWorker = true` → flow follows to **SomeWorker**, skipping LastThing.
* If `goToSomeWorker = false` → flow follows to **DoLastThing**, skipping SomeWorker.

---

👉 That’s it — all done through **drag, drop, and property edits in Camunda Modeler**.

---

Would you like me to also give you a **mini checklist of what to test in Operate** after making this change, so you can confirm both paths behave correctly?
