Excellent clarification!
Here is a **detailed, production-ready, and fully annotated plan** for a **support ticket assignment system** when:

* **All internal users have `role: 'user' | 'admin' | 'superadmin'`**
* **Department (e.g., 'support', 'IT', 'HR', 'Finance') is just a String field** (chosen from a fixed list like `['support', 'IT', ...]`)
* **Department list is defined in code or config, not in DB**
* **Assignment is based on department matching (not a support agent role)**
* **All users in a department can be assigned tickets for that department**
* **Assignment must be fair (least-loaded), with admin override**
* **External users only create tickets, not assigned to them**

---

# Department-based Support Ticket Assignment (No Agent Role, Static Departments)

---

## **1. System Overview**

* **Users:**

  * role: 'user', 'admin', 'superadmin'
  * department: String, must match from the list of allowed departments
* **Departments:**

  * List: e.g., `['support', 'IT', 'HR', 'Finance', 'Admissions']`
  * This list is maintained in app config, code, or ENV, not DB.
* **Ticket assignment:**

  * Any user in the same department (and active) can be assigned a ticket for that department.
  * Admins and superadmins can be assigned any ticket.
  * Tickets are assigned to the least-loaded eligible user in the department.
  * If no user in the department, ticket is flagged for admin/superadmin manual assignment.

---

## **2. Available Departments**

```js
// config/departments.js
export const DEPARTMENTS = [
  'support', 'IT', 'HR', 'Finance', 'Admissions'
];
```

---

## **3. Internal User Model**

```js
import mongoose from 'mongoose';

const userSchema = new mongoose.Schema({
  name: { type: String, required: true, trim: true },
  email: { type: String, required: true, unique: true, lowercase: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin', 'superadmin'], default: 'user', required: true },
  department: { type: String, trim: true, required: true }, // Must be from DEPARTMENTS
  isActive: { type: Boolean, default: true },
  // ...other fields
});
userSchema.index({ email: 1 }, { unique: true });
userSchema.index({ department: 1, role: 1, isActive: 1 });

const User = mongoose.model('User', userSchema);
export default User;
```

* Department is **required** for assignment eligibility.

---

## **4. Support Ticket Model**

```js
import mongoose from 'mongoose';

const supportTicketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  message: { type: String, required: true },
  department: { type: String, required: true }, // Must be in DEPARTMENTS
  status: {
    type: String,
    enum: ['Open', 'Assigned', 'Pending Assignment', 'In Progress', 'Resolved', 'Closed'],
    default: 'Open'
  },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User', default: null },
  requester: {
    name: String,
    email: String,
    role: String
  },
  history: [
    {
      action: String,
      performedBy: String,
      timestamp: { type: Date, default: Date.now }
    }
  ],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

supportTicketSchema.index({ department: 1, status: 1 });

const SupportTicket = mongoose.model('SupportTicket', supportTicketSchema);
export default SupportTicket;
```

---

## **5. Assignment Logic**

### **Algorithm Steps:**

1. When a ticket is created, try to assign it immediately.
2. Find **all active users** in that ticket's department (from User model, not from a "support agent" role).

   * Include admins/superadmins **if you want them to be eligible for any department**:

     ```js
     // To include all admins/superadmins for any department, add them to eligible pool.
     ```
3. For each eligible user, count the number of tickets assigned to them that are still `Open`, `Assigned`, or `In Progress`.
4. Assign ticket to the eligible user with the **fewest such tickets**.
5. If no user is eligible, set ticket to `'Pending Assignment'` and notify admin/superadmin for manual assignment.
6. Log all assignments in ticket's `history`.

---

## **6. Sample Assignment Function (Production-ready)**

```js
import User from '../models/User.js';
import SupportTicket from '../models/SupportTicket.js';
import { DEPARTMENTS } from '../config/departments.js';

// Returns updated ticket, or null if not assigned
export async function assignTicket(ticket) {
  if (!DEPARTMENTS.includes(ticket.department)) {
    ticket.status = 'Pending Assignment';
    ticket.history.push({
      action: `Invalid department (${ticket.department})`,
      performedBy: 'system',
      timestamp: new Date()
    });
    await ticket.save();
    return ticket;
  }

  // Find all eligible users in this department (isActive)
  let eligibleUsers = await User.find({
    department: ticket.department,
    isActive: true
  });

  // Optionally: add all admins/superadmins to eligible pool for every ticket
  const extraAdmins = await User.find({
    role: { $in: ['admin', 'superadmin'] },
    isActive: true
  });
  // Filter out duplicates
  const allEligibleUserIds = [
    ...new Set([...eligibleUsers.map(u => String(u._id)), ...extraAdmins.map(u => String(u._id))])
  ];
  eligibleUsers = await User.find({ _id: { $in: allEligibleUserIds } });

  if (!eligibleUsers.length) {
    ticket.status = 'Pending Assignment';
    ticket.history.push({
      action: `No eligible users for department ${ticket.department}`,
      performedBy: 'system',
      timestamp: new Date()
    });
    await ticket.save();
    // TODO: Notify admin/superadmin (email, dashboard, etc)
    return ticket;
  }

  // Find user with least number of open/assigned/in progress tickets
  let minCount = Infinity, assignedUser = null;
  for (const user of eligibleUsers) {
    const count = await SupportTicket.countDocuments({
      assignedTo: user._id,
      status: { $in: ['Open', 'Assigned', 'In Progress'] }
    });
    if (count < minCount) {
      minCount = count;
      assignedUser = user;
    }
  }

  ticket.assignedTo = assignedUser._id;
  ticket.status = 'Assigned';
  ticket.history.push({
    action: `Assigned to ${assignedUser.name} (${assignedUser.email})`,
    performedBy: 'system',
    timestamp: new Date()
  });
  await ticket.save();
  // Optionally: notify the assigned user (email, dashboard notification)
  return ticket;
}
```

---

## **7. Ticket Creation Endpoint Example**

```js
// POST /api/support-tickets
import SupportTicket from '../models/SupportTicket.js';
import { assignTicket } from '../utils/assignment.js';
import { DEPARTMENTS } from '../config/departments.js';

export const createTicket = async (req, res) => {
  const { subject, message, department, name, email, role } = req.body;

  // Validate department
  if (!DEPARTMENTS.includes(department)) {
    return res.status(400).json({ error: 'Invalid department selected' });
  }

  const ticket = await SupportTicket.create({
    subject,
    message,
    department,
    status: 'Open',
    requester: { name, email, role },
    history: [
      { action: 'Ticket created', performedBy: name, timestamp: new Date() }
    ]
  });

  // Attempt automatic assignment
  await assignTicket(ticket);

  res.status(201).json(ticket);
};
```

---

## **8. Manual Assignment (Admin/Superadmin)**

```js
// PATCH /api/support-tickets/:id/assign
import User from '../models/User.js';
import SupportTicket from '../models/SupportTicket.js';

export const manualAssign = async (req, res) => {
  const { id } = req.params;
  const { userId } = req.body; // ID of the user to assign the ticket to

  const ticket = await SupportTicket.findById(id);
  if (!ticket) return res.status(404).json({ error: 'Ticket not found' });

  const user = await User.findById(userId);
  if (!user || !user.isActive) return res.status(400).json({ error: 'User not eligible' });

  // Optional: You may restrict manual assignment to admins/superadmins, or allow any department user
  ticket.assignedTo = user._id;
  ticket.status = 'Assigned';
  ticket.history.push({
    action: `Manually assigned to ${user.name} (${user.email})`,
    performedBy: req.user.name, // the admin assigning
    timestamp: new Date()
  });

  await ticket.save();
  res.json(ticket);
};
```

---

## **9. Dashboard/API Flow**

### **Typical GET endpoints:**

* **List all tickets:**
  `GET /api/support-tickets`
  (with filters: by department, by status, by assignedTo, etc)

* **Tickets for a department:**
  `GET /api/support-tickets?department=support`

* **Tickets assigned to user:**
  `GET /api/support-tickets?assignedTo=<userId>`

* **Tickets pending assignment:**
  `GET /api/support-tickets?status=Pending%20Assignment`

* **All available users in a department:**
  `GET /api/users?department=support&isActive=true`

---

## **10. Notes & Best Practices**

* **Department list should be validated on ticket/user creation and updates.**
* **Admin/superadmin dashboards** can use these endpoints to monitor all tickets, especially `"Pending Assignment"`.
* **All assignment actions** should be tracked in `history` array for auditability.
* **Notifications:** Send email/in-app alerts when a ticket is assigned or needs manual intervention.
* **Index your collections** by `department`, `assignedTo`, `status` for scaling.

---

## **11. Multi-Department (Optional, Advanced)**

If you want a user to be eligible for multiple departments, you can make:

```js
department: { type: [String], required: true }
```

and change queries to:

```js
User.find({ department: ticket.department, isActive: true });
```

or

```js
User.find({ department: { $in: [ticket.department] }, isActive: true });
```

---

## **12. Security**

* All mutation endpoints (`PATCH`, `POST`) should be protected by authentication middleware.
* Only admins/superadmins can assign manually or override assignments.
* Users may only view their own assigned tickets (unless admin/superadmin).

---

## **13. Full Assignment Flow Recap**

```plaintext
Ticket created (department set, from allowed list)
      ↓
Find all active users in department (+ admins/superadmins)
      ↓
If one or more eligible:
    Assign to least-loaded user
    Log assignment in ticket.history
    Notify user
Else:
    Mark ticket as 'Pending Assignment'
    Log event in ticket.history
    Notify admin/superadmin for manual assignment
```

---

**This system provides a scalable, fair, and fully auditable ticket assignment structure with static department logic and no need for a dedicated "agent" role.
If you want additional code samples for authentication/authorization, dashboard UIs, batch assignment, analytics, or Slack/email integration, let me know!**

---

This solution, with comments and code, is easily 1000+ lines in full backend implementation. If you want this split into multiple files and ready to run, ask for a ZIP or multi-file breakdown.
