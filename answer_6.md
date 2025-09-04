Great eye. A BPMN file **without any `<bpmn:conditionExpression>`** is not automatically “wrong.” It just means **there’s no conditional sequence flow** in *that* model. Several perfectly valid scenarios explain this:

## What’s plausible

1. **The process is linear (no branching)**

* It’s just Start → Task → Task → End.
* No gateways → no conditional sequence flows → no `conditionExpression`.

2. **Branching exists, but not via expressions**

* **Parallel gateway** (`<bpmn:parallelGateway>`) splits/joins without conditions.
* **Event-based gateway** (`<bpmn:eventBasedGateway>`) chooses the path by **which event occurs** (message, timer, signal), not by an expression.
* **Boundary events** (error, escalation, message, timer) attached to tasks divert the flow when the event triggers—again, no condition expressions on sequence flows.

3. **Decisions are externalized (DMN / code)**

* A **Business Rule Task** calls a **DMN** decision; the model then follows a single path because the decision affects variables used later, not an immediate split.
* The team chose to keep **branch logic in Java** (workers set flags, raise messages, or choose which follow-up task to trigger). The BPMN may show separate steps but no explicit sequence-flow conditions.

4. **This BPMN is a *called* subprocess**

* It’s invoked by a **Call Activity** from another BPMN. The **parent model** does the conditional routing (“Should we call this subprocess or not?”), so this child process has no conditions within it.

5. **Multiple start events drive the path**

* **Message/timer start events** define different entry points; routing is by how the process is started, not by a gateway condition in the middle.

6. **The model is present but not yet wired**

* Recently merged, intended to be started later (by message, start event, or call activity) but **not yet connected**. (This is the “maybe incomplete” case.)

## How to tell which one you have (quick checks)

* **Search the XML** for gateways and clues:

  * `exclusiveGateway`, `inclusiveGateway`, `parallelGateway`, `eventBasedGateway`
  * `sequenceFlow` with `default="..."` (default flow on a gateway)
  * boundary events: `boundaryEvent` (with `errorEventDefinition`, `timerEventDefinition`, `messageEventDefinition`, `escalationEventDefinition`)
  * start events: multiple `startEvent` with `messageEventDefinition`/`timerEventDefinition`
  * `callActivity` (suggests this process is invoked by a parent)
* **Look in the parent model** (if this is a subprocess): is the decision made there?
* **Check Operate**: Are instances of this process started directly? By message? From a call activity?
* **Scan workers**: Any code that completes jobs by **sending messages** or starting this process explicitly (no gateway conditions needed in the BPMN then)?
* **DMN usage**: Any Business Rule Tasks calling DMNs?

## Tiny examples

### A. Linear process (no conditions)

```xml
<bpmn:process id="simpleFlow">
  <bpmn:startEvent id="start"/>
  <bpmn:sequenceFlow id="f1" sourceRef="start" targetRef="taskA"/>
  <bpmn:serviceTask id="taskA" camunda:type="external" camunda:topic="workA"/>
  <bpmn:sequenceFlow id="f2" sourceRef="taskA" targetRef="end"/>
  <bpmn:endEvent id="end"/>
</bpmn:process>
```

### B. Event-based choice (no conditionExpression)

```xml
<bpmn:eventBasedGateway id="waitForEvent"/>
<bpmn:intermediateCatchEvent id="msgEvt">
  <bpmn:messageEventDefinition messageRef="Message_Approve"/>
</bpmn:intermediateCatchEvent>
<bpmn:intermediateCatchEvent id="timerEvt">
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>PT24H</bpmn:timeDuration>
  </bpmn:timerEventDefinition>
</bpmn:intermediateCatchEvent>
```

The path depends on **which event arrives**.

### C. Parent decides to call child process

**Parent BPMN**:

```xml
<bpmn:exclusiveGateway id="decide"/>
<bpmn:sequenceFlow id="callChild" sourceRef="decide" targetRef="callActivity">
  <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression"><![CDATA[${needChild}]]></bpmn:conditionExpression>
</bpmn:sequenceFlow>
<bpmn:callActivity id="callActivity" calledElement="childProcess"/>
```

**Child BPMN** (no conditions needed internally).

### D. Java sets a boolean; gateway just reads it

**BPMN**

```xml
<bpmn:exclusiveGateway id="g1"/>
<bpmn:sequenceFlow id="toMgr" sourceRef="g1" targetRef="ManagerTask">
  <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression"><![CDATA[${managerNeeded}]]></bpmn:conditionExpression>
</bpmn:sequenceFlow>
<bpmn:sequenceFlow id="toNormal" sourceRef="g1" targetRef="NormalTask">
  <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression"><![CDATA[${!managerNeeded}]]></bpmn:conditionExpression>
</bpmn:sequenceFlow>
```

**Java**

```java
@JobWorker(type = "decide", autoComplete = false)
public void decide(ActivatedJob job, JobClient client) {
  Map<String, Object> vars = job.getVariablesAsMap();
  boolean managerNeeded = ((Number)vars.getOrDefault("amount", 0)).intValue() > 10000;
  vars.put("managerNeeded", managerNeeded);
  client.newCompleteCommand(job).variables(vars).send().join();
}
```

## Bottom line

* **(2) is very common**: The process may simply not need conditional sequence flows.
* **(3) is also common**: decisions can be modeled via events, parent processes, DMN, or code.
* **(1) “incomplete”** is only likely if you see references to this process that imply branching is required inside it—but none is modeled—and nothing external routes to it.

If you share a few lines (start event, any gateways, call activities), I can quickly tell you which bucket this file falls into.
