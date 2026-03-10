# Exercise 2+ (10 min): Saga Compensation for E-commerce Orders - Facilitation Guide

## Facilitation Guide

### Timing

* 1 min: restate scenario and constraints
* 4 min: Part A
* 3 min: Part B
* 2 min: Part C + quick share

### Prompts to Ask

* "What exact state means compensation is complete?"
* "How do you avoid double-voiding payment on duplicate events?"
* "When do you escalate to manual intervention instead of retrying forever?"

### Common Mistakes

* No explicit terminal compensation state
* Compensation commands not idempotent
* Duplicate `InventoryFailed` triggers duplicate side effects
* No decision rule for unresolved timeout uncertainty

---

