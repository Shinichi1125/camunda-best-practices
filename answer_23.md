Great — let’s dig into **variables** in Operate. This is often where teams get tripped up with Call Activities.

---

## 🔹 How variables move across a Call Activity

* Parent process (`somethingRelated`) calls child process (`anotherProcess`) using a **Call Activity**.
* By default in Zeebe (Camunda 8):

  * If `propagateAllChildVariables="true"` → *all parent variables* are copied into the child.
  * If `propagateAllChildVariables="false"` → only explicitly mapped variables are passed (or none, if no mapping is configured).
* In your case:

  ```xml
  <zeebe:calledElement processId="Process_anotherProcess" propagateAllChildVariables="false" />
  ```

  So you should expect **only mapped variables** (configured in Modeler under “Input Mappings”).

---

## 🔹 How to check in Operate

1. **Run the parent process**

   * Trigger a new instance of `somethingRelated`.

2. **Open the parent instance in Operate**

   * Go to `http://localhost:8080/operate`.
   * Find your running/finished `somethingRelated` instance.

3. **Locate the Call Activity**

   * Click on the Call Activity node (the task with the **small plus (+)** marker).
   * On the right-hand side, you’ll see details.

4. **Expand “Called Process Instance”**

   * Operate shows a section for called process instances.
   * Click the child process instance (`Process_anotherProcess`) to open it.

5. **Inspect Variables in the Child**

   * Once you’re inside `anotherProcess`:

     * On the **left-hand panel**, open the **Variables** tab.
     * You’ll see a JSON view of all variables available in the child.
   * Compare these to the parent variables:

     * Go back to the parent instance.
     * Open its Variables tab.

6. **Match against mappings**

   * If you see fewer variables in the child:
     That’s because `propagateAllChildVariables="false"`. Only mapped ones crossed over.
   * If nothing passed:
     That means no mappings were configured in Modeler for that Call Activity.

---

## 🔹 Example

### Parent variables in `somethingRelated`

```json
{
  "fundId": "F123",
  "caseId": "C456",
  "amount": 20000,
  "goToAnotherProcess": true
}
```

### Child variables in `anotherProcess` (with propagateAllChildVariables="false")

```json
{
  "fundId": "F123",
  "caseId": "C456"
}
```

So only `fundId` and `caseId` made it through, because they were explicitly mapped in the Call Activity.

---

✅ **Summary:**
In Operate → open parent → click Call Activity → open child process → inspect Variables tab. That shows exactly what data `anotherProcess` received, and lets you verify whether mappings/flags are behaving as expected.

---

Would you like me to also show you **where in Camunda Modeler** you configure those variable mappings for the Call Activity (the UI steps)? That way you can see why Operate is showing what it’s showing.
