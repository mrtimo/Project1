# BPMN Snippet Modeling Quiz

Model each scenario using **2–3 BPMN elements**. Focus on correct symbol choice and connection.

---

## Events

**Q1.** A customer submits an order online. This triggers the start of an order-processing process, which ends when the order is confirmed.

> Model: the start of the process and the end of the process.

---

**Q2.** Every night at midnight, a batch job automatically begins running to generate daily sales reports.

> Model: how the process is triggered and the first activity it performs.

---

**Q3.** A support agent resolves a customer complaint. Upon resolution, the process ends by sending a notification email to the customer.

> Model: the final activity and the type of end event used.

---

**Q4.** During order fulfilment, the warehouse system pauses and waits to receive a payment-confirmed message from the finance system before continuing to pick and pack the goods.

> Model: the waiting point and the activity that follows.

---

## Gateways

**Q5.** After a loan application is reviewed, it is either approved or rejected. Each outcome leads to a different next step, and only one outcome is possible.

> Model: the decision point and the two outgoing paths (label each path).

---

**Q6.** Once a project kick-off meeting is completed, two workstreams begin at the same time: the design team starts creating wireframes, and the development team begins setting up the environment.

> Model: the split point and the two parallel activities.

---

**Q7.** After evaluating a marketing campaign, the team may send an email update, post on social media, or do both — depending on the budget available.

> Model: the gateway and two outgoing paths that may both be taken.

---

**Q8.** After dispatching an order, the process waits for one of two possible events: a "delivery confirmed" message arrives, or a 48-hour timer expires. Whichever happens first determines what happens next.

> Model: the gateway that routes based on which event occurs first.

---

## Activities & Boundary Events

**Q9.** A task involves processing a payment. If the payment takes longer than 30 minutes, the process should escalate along a separate path while the payment task is cancelled.

> Model: the task and the boundary event that triggers the escalation path.

---

**Q10.** A task involves calling an external API to retrieve customer data. If the API returns an error, the process must immediately follow an error-handling path rather than continuing normally.

> Model: the task and the boundary event that handles the error.

---

**Q11.** While a manager is reviewing a document, a message can arrive at any time to cancel the review and redirect the process to an urgent-handling path. The review task is interrupted when this happens.

> Model: the task and the boundary event triggered by the incoming message.

---

**Q12.** While a long-running data migration task is running, a warning notification should be sent every time a non-critical issue is detected — without stopping the migration itself.

> Model: the task and the boundary event that triggers the notification without interrupting the task.

---
