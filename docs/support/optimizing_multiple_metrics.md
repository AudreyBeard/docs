---
title: "Optimizing multiple metrics"
<<<<<<< HEAD
tags:
   - sweeps
---

=======
tags: []
---

### Optimizing multiple metrics
>>>>>>> 6ef6c45f4d73f19772f9c0cbb4fbccb2cee243c2
If you want to optimize multiple metrics in the same run, you can use a weighted sum of the individual metrics.

```python
metric_combined = 0.3 * metric_a + 0.2 * metric_b + ... + 1.5 * metric_n
wandb.log({"metric_combined": metric_combined})
```

Ensure to log your new combined metric and set it as the optimization objective:

```yaml
metric:
  name: metric_combined
  goal: minimize
```