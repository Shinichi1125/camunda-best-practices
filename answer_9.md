Great question 👍 — this is where the **separation between Java code and BPMN orchestration** becomes really important.

---

### 🔍 Key idea

In Camunda 8, **workers don’t call each other directly in Java code**.
Instead, they are triggered by **service tasks in BPMN**. The BPMN process model defines *what comes next*.

So:

* `PreviousWorker.java` does its work inside a **service task** in `PreviousProcess.bpmn`.
* At the end of that task, the **BPMN sequence flow** decides what happens:

  * If condition X → flow continues into the `SomeWorker` service task.
  * Else → flow continues into the “last thing” task.

It doesn’t matter that `PreviousWorker` and `SomeWorker` are in different Java classes/files — BPMN orchestrates which worker is called next.

---

### ✅ What happens in your example

1. `PreviousWorker` runs.
2. BPMN reaches an **exclusive gateway** or conditional sequence flow.
3. If condition is met → BPMN activates the `SomeWorker` job.

   * “Last thing” is **not executed**, because the process didn’t follow that sequence flow.
4. If condition is not met → BPMN skips `SomeWorker` and goes to “last thing”.

So yes — the “last thing” is skipped **if BPMN says so**, not because the workers are in separate files.

---

### 🔧 Example BPMN (PreviousProcess.bpmn)

```xml
<bpmn:process id="PreviousProcess" isExecutable="true">
  <bpmn:startEvent id="Start"/>
  <bpmn:serviceTask id="DoSomething" name="Do Something" camunda:type="external" camunda:topic="doSomething"/>
  <bpmn:serviceTask id="DoAnotherThing" name="Do Another Thing" camunda:type="external" camunda:topic="doAnotherThing"/>
  <bpmn:exclusiveGateway id="Decision"/>

  <!-- If condition true → go to SomeWorker -->
  <bpmn:sequenceFlow id="toSomeWorker" sourceRef="Decision" targetRef="SomeWorkerTask">
    <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression"><![CDATA[${goToSomeWorker}]]></bpmn:conditionExpression>
  </bpmn:sequenceFlow>

  <!-- Else → go to LastThing -->
  <bpmn:sequenceFlow id="toLastThing" sourceRef="Decision" targetRef="LastThingTask">
    <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression"><![CDATA[${!goToSomeWorker}]]></bpmn:conditionExpression>
  </bpmn:sequenceFlow>

  <bpmn:serviceTask id="SomeWorkerTask" name="Some Worker" camunda:type="external" camunda:topic="somePlan"/>
  <bpmn:serviceTask id="LastThingTask" name="Last Thing" camunda:type="external" camunda:topic="lastThing"/>
  <bpmn:endEvent id="End"/>
</bpmn:process>
```

---

### ✅ Corresponding workers

`PreviousWorker.java`:

```java
@JobWorker(type = "doAnotherThing", autoComplete = false)
public void doAnotherThing(final ActivatedJob job, final JobClient client) {
    Map<String, Object> vars = new HashMap<>(job.getVariablesAsMap());

    // your logic
    boolean condition = ...;
    vars.put("goToSomeWorker", condition);

    client.newCompleteCommand(job).variables(vars).send().join();
}
```

`SomeWorker.java`:

```java
@JobWorker(type = "somePlan", autoComplete = false)
public void somePlan(final ActivatedJob job, final JobClient client) {
    ...
}
```

`LastThingWorker.java`:

```java
@JobWorker(type = "lastThing", autoComplete = false)
public void lastThing(final ActivatedJob job, final JobClient client) {
    ...
}
```

---

### TL;DR

* Workers in different files **don’t automatically “skip” each other**.
* The **BPMN model decides** if the process moves from `PreviousWorker` → `SomeWorker` or `PreviousWorker` → `LastThing`.
* So yes: if BPMN directs the flow into `SomeWorker`, the “last thing” will be skipped, no matter that the Java classes are separate.

---

Do you want me to also sketch what it would look like if **`PreviousWorker` tried to call `SomeWorker` directly in Java (without BPMN)**, and why that’s usually *not recommended*?
