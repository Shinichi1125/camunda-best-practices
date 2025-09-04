Exactly — you spotted a classic **floating-point precision** issue! In Java (and most languages), `double` is a **binary floating-point** type. It can’t precisely represent many decimal fractions, so calculations like `755.33 - 405.06` can result in `350.27000000000004`.

For **money or precise decimal arithmetic**, `double` is **not recommended**.

---

### 🔥 Best practice for money and exact decimals in Java

Use **`BigDecimal`** instead of `double`.

---

#### Why `BigDecimal` is better

* Represents numbers as **decimal** (not binary), avoiding precision errors.
* Allows explicit control over **rounding** and **scale** (number of decimal places).
* Safest choice for financial calculations, invoices, currency amounts, etc.

---

#### Example

```java
import java.math.BigDecimal;
import java.math.RoundingMode;

public class MoneyCalc {
    public static void main(String[] args) {
        BigDecimal a = new BigDecimal("755.33");
        BigDecimal b = new BigDecimal("405.06");

        BigDecimal result = a.subtract(b);

        // Round to 2 decimal places
        BigDecimal rounded = result.setScale(2, RoundingMode.HALF_UP);

        System.out.println("Result: " + rounded); // prints 350.27
    }
}
```

Key points:

* **Use `String` constructor** for `BigDecimal` to avoid inheriting double’s floating error.
* Always specify **rounding mode** when changing scale.

---

#### If values come as `double`

If you’re given a `double`, wrap it like this:

```java
BigDecimal bd = BigDecimal.valueOf(755.33);
```

`BigDecimal.valueOf()` is safer than `new BigDecimal(double)` because it avoids exposing the binary floating error.

---

### 🔍 When is `double` okay?

* Approximate calculations (graphics, physics simulations, scientific work).
* Not recommended for financial or legal-sensitive numbers.

---

### TL;DR

✅ For money and “clean” decimal math in Java:

* Use `BigDecimal` (with `String` or `valueOf()` constructors).
* Set `scale` and `RoundingMode` explicitly.
* Avoid `double` for monetary amounts.

---

Would you like me to show **how to refactor an existing class that currently uses `double` for money fields** into `BigDecimal` cleanly?
