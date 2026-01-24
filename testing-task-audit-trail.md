# Testing Guide: Task Audit Trail Feature

This guide covers manual testing of the task audit trail feature that tracks field edits, carry-forwards, and deletions.

## Prerequisites

1. Start the development server:
   ```bash
   npm run dev
   ```

2. Have test accounts ready:
   - **Admin**: admin@taskflow.com
   - **Manager**: Any manager account
   - **Employee**: Any employee account

---

## Test Cases

### Test 1: Field Edit Tracking

**Goal:** Verify that editing a task creates history entries visible in the History tab.

**Steps:**
1. Login as any user who owns a task
2. Navigate to **Tasks** page
3. Click on any task to open the **Task Detail Modal**
4. Note the current values (title, priority, etc.)
5. Close the modal
6. Edit the task (if edit UI exists) OR use Prisma Studio to simulate:
   ```bash
   npx prisma studio
   ```
   - Find the task in the `Task` table
   - Change the `title` field (e.g., add " - Updated" at the end)
   - Save

7. Manually create an edit history entry in `TaskEditHistory` table:
   - `taskId`: The task's ID
   - `editedById`: Your user ID
   - `fieldName`: "title"
   - `oldValue`: Original title
   - `newValue`: New title
   - `createdAt`: Current timestamp

8. Refresh the Tasks page
9. Click on the edited task
10. Go to **History** tab

**Expected Result:**
- You should see a **blue pencil icon** entry showing:
  - "[User Name] changed Title from [old value] to [new value]"
  - Timestamp of the change

---

### Test 2: Carry-Forward Display in History

**Goal:** Verify that carry-forward (deadline extension) entries appear in the History tab.

**Steps:**
1. Login as the task owner
2. Navigate to **Tasks** page
3. Find a task with a deadline that can be extended
4. Click the **Carry Forward** button (or equivalent)
5. Enter a new deadline and reason (minimum 10 characters)
6. Submit the carry-forward request
7. Open the task again
8. Go to **History** tab

**Expected Result:**
- You should see an **orange calendar icon** entry showing:
  - "[User Name] extended deadline from [old date] to [new date]"
  - The reason provided
  - Timestamp of the extension

---

### Test 3: Multiple Activity Types in Timeline

**Goal:** Verify that the History tab shows all activity types in chronological order.

**Steps:**
1. Login as a manager or admin
2. Find or create a task
3. Perform multiple actions on the task:
   - **Action 1:** Accept the task (status change)
   - **Action 2:** Start the task (status change)
   - **Action 3:** Carry forward the deadline
   - **Action 4:** Edit the title (if edit UI available)
   - **Action 5:** Put on hold with reason
4. Open the task and go to **History** tab

**Expected Result:**
- All activities appear in **reverse chronological order** (newest first)
- Each activity type has a distinct icon:
  | Icon | Color | Activity Type |
  |------|-------|---------------|
  | Clock/History | Gray | Status changes |
  | Calendar | Orange | Carry-forwards |
  | Pencil | Blue | Field edits |

---

### Test 4: Deletion Tracking (Database Verification)

**Goal:** Verify that deleting a task records who deleted it.

**Steps:**
1. Login as task owner or admin
2. Create a new test task (or use an existing one you can delete)
3. Note the task ID (visible in URL or Prisma Studio)
4. Delete the task using the **Delete** button in Task Detail Modal
5. Open Prisma Studio:
   ```bash
   npx prisma studio
   ```
6. Find the deleted task in the `Task` table (filter by the ID)

**Expected Result:**
- `deletedAt` should have a timestamp
- `deletedById` should contain your user ID (not null)

---

### Test 5: History Tab Visibility by Role

**Goal:** Verify that users can only see history for tasks they have access to.

**Steps:**

#### As Employee:
1. Login as employee
2. Open one of YOUR tasks
3. Go to History tab
4. **Expected:** Can see full history of your own tasks

#### As Manager:
1. Login as manager
2. Open a subordinate's task
3. Go to History tab
4. **Expected:** Can see full history of subordinate's tasks

#### As Admin:
1. Login as admin
2. Open any user's task
3. Go to History tab
4. **Expected:** Can see full history of all tasks

---

## Database Verification (Optional)

### Check Edit History Entries
```bash
npx prisma studio
```
Navigate to `TaskEditHistory` table and verify:
- `taskId` links to the correct task
- `editedById` links to the user who made the change
- `fieldName` shows which field was changed
- `oldValue` and `newValue` contain the before/after values
- `createdAt` has the correct timestamp

### Check Deletion Tracking
Navigate to `Task` table and filter for deleted tasks:
- `deletedAt` should have a timestamp
- `deletedById` should have a user ID

---

## Troubleshooting

### History tab shows "No history yet"
1. Verify the task has status changes (at minimum, task creation)
2. Check browser console for API errors
3. Verify `/api/tasks/[taskId]` response includes `statusHistory`, `carryForwardLogs`, `editHistory`

### Carry-forward entries not showing
1. Verify the task was actually carried forward (check `isCarriedForward` field)
2. Check `CarryForwardLog` table in Prisma Studio
3. Ensure the carry-forward was successful (not just attempted)

### Edit history not showing
1. Verify edits were made through the API (not direct DB changes without history)
2. Check `TaskEditHistory` table in Prisma Studio
3. Note: Only edits made AFTER this feature was deployed will be tracked

### Delete doesn't record actor
1. Verify you're using the Delete button in the UI (not direct DB deletion)
2. Check that the API call goes through `/api/tasks/[taskId]` DELETE endpoint

---

## Quick Checklist

| Feature | What to Check | Where to Verify |
|---------|---------------|-----------------|
| Status History | Gray icon entries | History tab |
| Carry-Forward | Orange icon entries | History tab |
| Field Edits | Blue icon entries | History tab |
| Timeline Order | Newest first | History tab |
| Delete Actor | `deletedById` populated | Prisma Studio > Task table |

---

## Notes

- **Edit tracking** only works for edits made through the PUT `/api/tasks/[taskId]` endpoint
- **Tracked fields:** title, description, priority, size, estimatedMinutes, actualMinutes, deadline, startDate, kpiBucketId
- **Carry-forward logs** existed before this update; now they're displayed in the History tab
- **Deletion tracking** is backend-only; there's no UI to see who deleted a task (query the database)
