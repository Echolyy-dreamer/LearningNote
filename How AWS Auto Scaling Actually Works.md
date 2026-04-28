# Study Note: How AWS Auto Scaling Actually Works

## 1. The Universal Workflow (Three Phases)
All scaling policies follow the same lifecycle: **Observation → Decision → Action**.
- Metric Observation: EC2 sends telemetry including CPU, network and custom metrics to CloudWatch.
- Decision Logic: Scaling policies evaluate system status based on metric data.
- Action Execution: EC2 Auto Scaling adjusts capacity via Launch Template and updates ALB Target Group.

## 2. Target Tracking (Goal-Oriented Model)
A declarative, set-and-forget model to maintain a steady target, such as 50% average CPU.
- AWS creates hidden, system-managed CloudWatch alarms prefixed with `TargetTracking-`.
- It recalculates required capacity dynamically to narrow the gap between real-time metric and target.
- Proportional control logic:
$$
\text{New Capacity} \approx \text{Current Capacity} \times \frac{\text{Current Metric}}{\text{Target Value}}
$$
- Supports large-scale scaling in a single action.
- Core fact: Target Tracking is inherently **alarm-driven** with invisible AWS-managed alarms.

## 3. Step Scaling (Rule-Based Model)
An imperative, event-driven model with manually defined thresholds and fixed actions.
1. CloudWatch collects metrics.
2. Users create visible custom CloudWatch Alarms.
3. Scaling triggers when alarms enter ALARM state.
4. ASG executes fixed incremental actions, like adding 2 instances.

## 4. Core Difference
| Feature | Step Scaling | Target Tracking |
|---------|--------------|-----------------|
| Alarm Ownership | User-defined & manual | System-managed & automated |
| Visibility | Visible in CloudWatch | Hidden in console |
| Trigger Model | Threshold event-driven | Continuous state evaluation |
| Scaling Behavior | Fixed step increments | Dynamic proportional adjustment |

## 5. Fact Check: Polling vs. Alarms
- Misconception: ASG continuously polls CloudWatch metrics.
- Reality: All scaling actions are triggered by **CloudWatch Alarm state transitions**, not direct polling.

## 6. Why Target Tracking Hides Alarms
- Lower operational complexity of threshold tuning.
- Prevent scaling flapping and unstable oscillation.
- Shift from imperative trigger management to declarative goal-based management.

## 7. Summary
- Unified workflow for all ASG: Metric → Decision → Action.
- Step Scaling: White-box, user-controlled alarms.
- Target Tracking: Black-box, AWS-managed implicit alarms.
- Both are alarm-driven at the infrastructure layer.
- Target Tracking simplifies operation by focusing on business goals rather than underlying controls.
