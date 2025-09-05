Perfect 👍 — let’s compare **before vs. after** in both a **visual sketch** and a **BPMN XML snippet** so it’s very concrete.

---

# 🔹 Before (current flow)

### Visual sketch

```
(Start)
   |
Do Something
   |
Do Another Thing
   |
Do Last Thing   ---> SomeWorker (conditionally, but Last Thing always runs first)
   |
 (End)
```

So *Last Thing is always executed*, and after that the process may also enter SomeWorker.

### BPMN XML (simplified)

```xml
<bpmn:serviceTask id="DoAnotherThing" name="Do Another Thing"
                  camunda:type="external" camunda:topic="doAnotherThing" />

<bpmn:serviceTask id="DoLastThing" name="Do Last Thing"
                  camunda:type="external" camunda:topic="doLastThing" />

<bpmn:serviceTask id="SomeWorkerTask" name="Some Worker"
                  camunda:type="external" camunda:topic="somePlan" />

<!-- Flows -->
<bpmn:sequenceFlow id="flow1" sourceRef="DoAnotherThing" targetRef="DoLastThing"/>
<bpmn:sequenceFlow id="flow2" sourceRef="DoLastThing" targetRef="SomeWorkerTask">
  <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression"><![CDATA[${goToSomeWorker}]]></bpmn:conditionExpression>
</bpmn:sequenceFlow>
```

---

# 🔹 After (desired flow)

### Visual sketch

```
(Start)
   |
Do Something
   |
Do Another Thing
   |
  [Exclusive Gateway]
    /         \
  yes          no
  /             \
SomeWorker    Do Last Thing
   |
 (End)
```

So now, *either SomeWorker OR Last Thing* runs, but not both.

### BPMN XML (simplified)

```xml
<bpmn:serviceTask id="DoAnotherThing" name="Do Another Thing"
                  camunda:type="external" camunda:topic="doAnotherThing" />

<bpmn:exclusiveGateway id="Decision" name="Check Condition" />

<bpmn:serviceTask id="DoLastThing" name="Do Last Thing"
                  camunda:type="external" camunda:topic="doLastThing" />

<bpmn:serviceTask id="SomeWorkerTask" name="Some Worker"
                  camunda:type="external" camunda:topic="somePlan" />

<!-- Flows -->
<bpmn:sequenceFlow id="flow1" sourceRef="DoAnotherThing" targetRef="Decision"/>

<bpmn:sequenceFlow id="toSomeWorker" sourceRef="Decision" targetRef="SomeWorkerTask">
  <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression"><![CDATA[${goToSomeWorker}]]></bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="toLastThing" sourceRef="Decision" targetRef="DoLastThing">
  <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression"><![CDATA[${!goToSomeWorker}]]></bpmn:conditionExpression>
</bpmn:sequenceFlow>
```

---

# 🔹 Java Worker support

In `PreviousWorker.java`, you just need to set the variable `goToSomeWorker` when completing the job:

```java
@JobWorker(type = "doAnotherThing", autoComplete = false)
public void doAnotherThing(final ActivatedJob job, final JobClient client) {
    Map<String, Object> vars = new HashMap<>(job.getVariablesAsMap());

    boolean condition = ...; // your business logic
    vars.put("goToSomeWorker", condition);

    client.newCompleteCommand(job).variables(vars).send().join();
}
```

---

✅ **Result:**

* Before: `DoLastThing` always runs, then *maybe* `SomeWorker`.
* After: Exclusive gateway chooses between `DoLastThing` **OR** `SomeWorker`.

---

Would you like me to also show how this looks in **Camunda Modeler’s UI** (screenshots or step-by-step clicks), so you could replicate the change visually instead of editing XML?
