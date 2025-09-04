love this prompt — here are two concrete, real-life mini-processes that fit your structure (Java `@JobWorker` methods + a BPMN per worker). They’re simple enough to read, but realistic enough to see how things hang together.

---

# Example 1: Expense reimbursement

## What it does

* **SomeProcess.bpmn**: initialize request → decide if manager approval needed → (possibly) ask manager → finalize request
* **AnotherProcess.bpmn**: when approved, finance posts the payment

## SomeWorker.java (two workers)

```java
@Component
public class SomeWorker {

  @Autowired ObjectMapper objectMapper;

  // 1) Initialize the expense (parse form payload, normalize values)
  @JobWorker(type = "initializeExpense", autoComplete = false)
  public void initializeExpense(final ActivatedJob job, final JobClient client) throws Exception {
    Map<String, Object> vars = new HashMap<>(job.getVariablesAsMap());

    // Example inputs (from a start form or upstream step)
    // amount: number, currency: string, employeeId: string, category: string
    double rawAmount = ((Number) vars.getOrDefault("amount", 0)).doubleValue();
    String currency = (String) vars.getOrDefault("currency", "EUR");
    String category = (String) vars.getOrDefault("category", "GENERAL");

    // Normalize & enrich
    vars.put("amount", rawAmount);
    vars.put("currency", currency);
    vars.put("isHighAmount", rawAmount >= 1000.0);
    vars.put("requiresReceipt", !"MEALS".equalsIgnoreCase(category));

    client.newCompleteCommand(job).variables(vars).send().join();
  }

  // 2) Decide the plan: route to manager or skip
  @JobWorker(type = "expenseRoutingPlan", autoComplete = false)
  public void somePlan(final ActivatedJob job, final JobClient client) {
    Map<String, Object> vars = new HashMap<>(job.getVariablesAsMap());

    boolean isHighAmount = Boolean.TRUE.equals(vars.get("isHighAmount"));
    boolean requiresReceipt = Boolean.TRUE.equals(vars.get("requiresReceipt"));
    boolean receiptAttached = Boolean.TRUE.equals(vars.get("receiptAttached"));

    // A simple Java decision; BPMN gateway can just read 'managerApprovalNeeded'
    boolean managerApprovalNeeded = isHighAmount || (requiresReceipt && !receiptAttached);
    vars.put("managerApprovalNeeded", managerApprovalNeeded);

    client.newCompleteCommand(job).variables(vars).send().join();
  }
}
```

### SomeProcess.bpmn (core bits)

```xml
<bpmn:process id="ExpenseProcess" name="Expense Process" isExecutable="true">
  <bpmn:startEvent id="Start"/>
  <bpmn:sequenceFlow id="f1" sourceRef="Start" targetRef="Initialize"/>
  <bpmn:serviceTask id="Initialize" name="Initialize Expense"
      camunda:type="external" camunda:topic="initializeExpense"/>
  <bpmn:sequenceFlow id="f2" sourceRef="Initialize" targetRef="RoutePlan"/>
  <bpmn:serviceTask id="RoutePlan" name="Compute Routing Plan"
      camunda:type="external" camunda:topic="expenseRoutingPlan"/>

  <bpmn:sequenceFlow id="toMgr" sourceRef="RoutePlan" targetRef="ManagerApproval">
    <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression"><![CDATA[${managerApprovalNeeded}]]></bpmn:conditionExpression>
  </bpmn:sequenceFlow>
  <bpmn:sequenceFlow id="toFinalize" sourceRef="RoutePlan" targetRef="Finalize">
    <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression"><![CDATA[${!managerApprovalNeeded}]]></bpmn:conditionExpression>
  </bpmn:sequenceFlow>

  <bpmn:userTask id="ManagerApproval" name="Manager approval"/>
  <bpmn:sequenceFlow id="f3" sourceRef="ManagerApproval" targetRef="Finalize"/>

  <bpmn:serviceTask id="Finalize" name="Finalize Expense"
      camunda:type="external" camunda:topic="finalizeExpense"/>
  <bpmn:sequenceFlow id="f4" sourceRef="Finalize" targetRef="End"/>
  <bpmn:endEvent id="End"/>
</bpmn:process>
```

## AnotherWorker.java (finance payout)

```java
@Component
public class AnotherWorker {

  @JobWorker(type = "finalizeExpense", autoComplete = false)
  public void finalizeExpense(final ActivatedJob job, final JobClient client) {
    Map<String, Object> vars = new HashMap<>(job.getVariablesAsMap());

    // Pretend to call finance API; set result flags
    boolean paid = true; // result from API
    vars.put("paymentStatus", paid ? "PAID" : "FAILED");

    client.newCompleteCommand(job).variables(vars).send().join();
  }
}
```

### AnotherProcess.bpmn

Often this is just a **service task** in the same process (as above).
If you want a *separate* process (e.g., a called subprocess for finance), that child process can be modeled without conditions, just a single service task `finalizeExpense`.

---

# Example 2: Customer onboarding (KYC)

## What it does

* **SomeProcess.bpmn**: collect info → run checks → if high risk → manual review; else → auto-approve
* **AnotherProcess.bpmn**: send welcome pack & create account in core banking

## SomeWorker.java (initialize + plan)

```java
@Component
public class SomeWorker {

  @JobWorker(type = "initializeKyc", autoComplete = false)
  public void initializeKyc(final ActivatedJob job, final JobClient client) {
    Map<String, Object> vars = new HashMap<>(job.getVariablesAsMap());
    // inputs: personId, pep (politically exposed), countryRisk, docQualityScore
    boolean pep = Boolean.TRUE.equals(vars.get("pep"));
    int countryRisk = ((Number) vars.getOrDefault("countryRisk", 0)).intValue();
    int docScore = ((Number) vars.getOrDefault("docQualityScore", 0)).intValue();

    vars.put("pep", pep);
    vars.put("countryRisk", countryRisk);
    vars.put("docQualityScore", docScore);
    client.newCompleteCommand(job).variables(vars).send().join();
  }

  @JobWorker(type = "kycDecisionPlan", autoComplete = false)
  public void somePlan(final ActivatedJob job, final JobClient client) {
    Map<String, Object> vars = new HashMap<>(job.getVariablesAsMap());
    boolean pep = Boolean.TRUE.equals(vars.get("pep"));
    int countryRisk = ((Number) vars.getOrDefault("countryRisk", 0)).intValue();
    int docScore = ((Number) vars.getOrDefault("docQualityScore", 0)).intValue();

    boolean needsManualReview = pep || countryRisk >= 3 || docScore < 75;
    vars.put("needsManualReview", needsManualReview);

    client.newCompleteCommand(job).variables(vars).send().join();
  }
}
```

### SomeProcess.bpmn (core bits)

```xml
<bpmn:process id="KycProcess" name="KYC" isExecutable="true">
  <bpmn:startEvent id="Start"/>
  <bpmn:sequenceFlow id="f1" sourceRef="Start" targetRef="InitializeKyc"/>
  <bpmn:serviceTask id="InitializeKyc" name="Initialize KYC"
      camunda:type="external" camunda:topic="initializeKyc"/>

  <bpmn:sequenceFlow id="f2" sourceRef="InitializeKyc" targetRef="KycPlan"/>
  <bpmn:serviceTask id="KycPlan" name="Compute KYC Decision"
      camunda:type="external" camunda:topic="kycDecisionPlan"/>

  <bpmn:exclusiveGateway id="g1" name="KYC Gateway"/>
  <bpmn:sequenceFlow id="toManual" sourceRef="KycPlan" targetRef="ManualReview">
    <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression"><![CDATA[${needsManualReview}]]></bpmn:conditionExpression>
  </bpmn:sequenceFlow>
  <bpmn:sequenceFlow id="toAuto" sourceRef="KycPlan" targetRef="AutoApprove">
    <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression"><![CDATA[${!needsManualReview}]]></bpmn:conditionExpression>
  </bpmn:sequenceFlow>

  <bpmn:userTask id="ManualReview" name="Compliance review"/>
  <bpmn:sequenceFlow id="f3" sourceRef="ManualReview" targetRef="AutoApprove"/>

  <bpmn:serviceTask id="AutoApprove" name="Create Account"
      camunda:type="external" camunda:topic="createAccount"/>
  <bpmn:endEvent id="End"/>
</bpmn:process>
```

## AnotherWorker.java (create account + welcome)

```java
@Component
public class AnotherWorker {

  @JobWorker(type = "createAccount", autoComplete = false)
  public void createAccount(final ActivatedJob job, final JobClient client) {
    Map<String, Object> vars = new HashMap<>(job.getVariablesAsMap());

    // Call core system; assume success
    String accountId = "ACC-" + System.currentTimeMillis();
    vars.put("accountId", accountId);
    vars.put("onboardingStatus", "APPROVED");

    client.newCompleteCommand(job).variables(vars).send().join();
  }
}
```

### AnotherProcess.bpmn

Again, this can be a single service task in the same process, or a called subprocess (`calledElement="CreateAccountProcess"`) if you want to separate responsibilities.

---

## Notes & tiny best-practices

* **autoComplete**: if `true`, return a `Map<String,Object>` from the worker to persist variables automatically; if `false`, call `client.newCompleteCommand(job).variables(vars).send().join()`.
* **Variable naming**: keep keys stable and documented (`managerApprovalNeeded`, `needsManualReview`, `paymentStatus`, etc.).
* **Keep BPMN simple**: let gateways read a **boolean flag** decided in Java (as you suggested earlier). This is easy for devs who aren’t BPMN experts.
* **Operate**: you’ll see each service task as a job, plus which branch was taken by the gateway.

If you want, I can drop both examples into a single ZIP-friendly layout (two BPMNs, two workers) so you can paste into your project and run.
