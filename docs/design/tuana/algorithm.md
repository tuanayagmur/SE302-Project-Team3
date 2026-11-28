# T3 – Scheduling Algorithm

This file explains how `ScheduleGenerator` places the exams in time slots and classrooms, and how `ScheduleValidator` checks the rules. Requirement numbers refer to the Requirements Document.

## 1. Input and output

**Input:** courses with their students, exam durations, difficulty levels and needed features; classrooms with capacities, features and unavailable slots; the exam period with its time slots and settings (changeover time); exams fixed by the user (FR 9).

**Output:** an `ExamSchedule` with one `ScheduledExam` per course. Each exam has one `TimeSlot` and one or more `ClassroomAllocation`s, and every student is seated in one classroom (FR 7, 28, 33, 34).

## 2. Rules

**Hard rules** (a schedule that breaks one of these is never saved, NFR 8):
1. Each exam is in exactly one time slot, and the slot is at least as long as the exam (FR 28, 29).
2. A student has no two exams in the same slot (FR 30).
3. A student has at most two exams on one day (FR 31).
4. A student has no exams in consecutive slots of the same day (FR 32).
5. The classrooms of an exam have enough total capacity (FR 33).
6. A classroom has one exam at a time, and there is at least the changeover time between two exams in the same classroom (FR 35, 36).
7. Unavailable classrooms are not used, and an exam that needs a feature only gets classrooms with that feature (FR 37, 38).

**Soft rules** (used to choose between valid options):
1. Spread each student's exams over more days (FR 41).
2. Place difficult exams of the same student further apart (FR 13).
3. Do not use computer labs for other exams when ordinary classrooms are free (FR 39).
4. After a data change, keep exams in their old slots (FR 42).

## 3. Steps

### Step 1 – Feasibility check: `checkFeasibility()`
Before scheduling, the generator looks for problems that make any schedule impossible and returns them as a list of messages (FR 8):
1. A course has more students than the total capacity of the classrooms suitable for it (FR 50).
2. An exam is longer than every time slot (FR 51).
3. The exam period is too short (FR 52). Because a student can have at most two exams per day and no exams in consecutive slots, the generator finds the student with the most exams and calculates how many days are needed for them.

If the list is not empty, the schedule is not created and the messages are shown to the user.

### Step 2 – Order the exams
Two courses **conflict** if they share at least one student. The generator counts the conflicts of every course and orders the courses so that the hardest ones are placed first:
1. Fixed exams (FR 9). They are placed in their chosen slot first.
2. Then by number of conflicting courses (most first).
3. Then by number of students (most first).

### Step 3 – Place each exam: `generate()`
For each course in this order:
1. **Find valid slots.** A slot is valid if hard rules 1–4 hold for all students of the course and step 4 can find classrooms for it in that slot.
2. **Choose the best slot.** Each valid slot gets a cost from the soft rules. The cost increases when a student already has an exam on that day, when two difficult exams of a student come close, and when the exam moves away from its old slot. The slot with the lowest cost is chosen.
3. **If there is no valid slot,** the generator tries a repair: it takes one already placed exam that blocks the slot and moves it to another valid slot. If that does not work, it reports which courses could not be placed (FR 8).

### Step 4 – Choose the classrooms
For an exam with `n` students in a chosen slot, only classrooms that are free in that slot, available, have the needed features and satisfy the changeover time (rule 6) are used:
1. If one classroom is large enough, the smallest such classroom is chosen. This leaves big classrooms for big exams.
2. Otherwise, the largest classrooms are taken one by one until their total capacity reaches `n`. The exam is therefore split across as few classrooms as possible (FR 33).
3. Computer labs are only used when the course needs them or no ordinary classroom is free (FR 39).
4. The students are sorted by student ID and seated in the classrooms in order. Each `ClassroomAllocation` gets its list of students (FR 34).

### Step 5 – Final check: `validate(s)`
`ScheduleValidator.validate(s)` checks all hard rules again on the finished schedule. If it finds a violation, the schedule is not saved and the problem is reported. This check makes sure that a mistake in the generator cannot produce a wrong schedule (NFR 8). After this, invigilators are assigned only if a list was imported (`InvigilatorAssigner`, FR 16).

## 4. Checking a manual move: `checkMove(e, t)`
When the user moves an exam by hand (FR 19), `checkMove(e, t)` uses the same checks as step 3 for the new slot `t`. It returns the broken rules and the affected students. The move is rejected if the list is not empty (FR 40).

## 5. Updating after a data change
When students or classrooms change, the schedule is marked as outdated (FR 26). `ScheduleUpdater` finds only the exams affected by the change (`findAffectedExams()`), for example exams with new students or exams in a classroom that was removed. It then calls the generator only for these exams. All other exams keep their slots and classrooms, and soft rule 4 keeps the affected exams close to their old slots (FR 42).

## 6. Performance and cancelling
1. The generator runs in a background thread and works on a copy of the data. The screen stays responsive and shows the progress (NFR 7).
2. If the user cancels, the copy is discarded and the saved data and schedule do not change (FR 53).
