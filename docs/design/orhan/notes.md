# Design Notes – T5 (Invigilator & Export)

## Scope
My parts in `class-list.md` are:
- `Invigilator` (FR 14, FR 15)
- `InvigilatorAssigner` (FR 16, FR 43–49)
- `ScheduleExporter` (FR 20, FR 21, FR 54)

Assigning invigilators is optional. If there is no invigilator list, the schedule still works and can be saved/exported without it.

---

## Classes (First Draft)

### `Invigilator`
Holds a staff member who supervises exams in a classroom (FR 14, FR 15, Appendix C).

**Fields:**
- `id: int` (matches `invigilator` table)
- `name: String`
- `unavailableSlots: List<TimeSlot>` (from `invigilator_unavailability`, FR 15)
- `assignedAllocations: List<ClassroomAllocation>` (from `allocation_invigilator`)

**Methods:**
- `isAvailable(slot: TimeSlot): boolean` – checks if the slot is not in `unavailableSlots` (FR 46)
- `addUnavailableSlot(slot: TimeSlot): void` – marks a slot as unavailable (FR 15)
- `removeUnavailableSlot(slot: TimeSlot): void`
- `getDailyDutyCount(date: LocalDate): int` – number of exams on that day (max 3 per day, FR 45)
- `getTotalDutyCount(): int` – total duties across the period (for balancing load, FR 48)

---

### `InvigilatorAssigner`
Assigns invigilators to each `ClassroomAllocation` when a list exists (FR 16, FR 43–49).

**Fields:**
- `studentsPerInvigilator: int` – default `40`, rounded up, configurable in `exam_period` (FR 43)
- `minBreakMinutes: int` – default `30` minutes between duties on the same day, configurable in `exam_period` (FR 47)

**Methods:**
- `assignInvigilators(schedule: ExamSchedule, invigilators: List<Invigilator>, period: ExamPeriod): AssignmentReport`
  - If `invigilators` is empty, does nothing and returns a report noting no list was provided.
  - Goes through each `ScheduledExam` and its `ClassroomAllocation` entries.
- `calculateRequiredCount(studentCount: int): int`
  - `ceil(studentCount / studentsPerInvigilator)` (FR 43).
- `isEligible(invigilator: Invigilator, exam: ScheduledExam, schedule: ExamSchedule): boolean`
  - No two classrooms in the same time slot (FR 44).
  - At most 3 exams on the same day (FR 45).
  - Not marked unavailable in that slot (FR 46).
  - At least `minBreakMinutes` gap between consecutive duties on the same day (FR 47).
- `selectBestCandidates(eligible: List<Invigilator>, count: int): List<Invigilator>`
  - Picks invigilators with the fewest total duties so the workload stays balanced (FR 48).
- `clearAssignments(schedule: ExamSchedule): void`
  - Clears old invigilator assignments before assigning again.

**`AssignmentReport`:**
- `success: boolean`
- `understaffedSlots: List<TimeSlot>` – slots where there weren't enough invigilators (FR 49)
- `shortageDetails: Map<TimeSlot, Integer>`

---

### `ScheduleExporter`
Exports and prints schedules and invigilation duties (FR 20, FR 21, FR 54).

**Methods:**
- `exportExamScheduleCsv(schedule: ExamSchedule, filePath: Path): void` – CSV with each exam's date, start/end time, classrooms, and invigilators if any (FR 20, FR 54)
- `exportStudentSchedulesCsv(schedule: ExamSchedule, students: List<Student>, filePath: Path): void` – CSV with each student's exams, date, start/end time, and assigned classroom (FR 54)
- `exportInvigilatorDutiesCsv(schedule: ExamSchedule, invigilators: List<Invigilator>, filePath: Path): void` – exports invigilator assignments (FR 20)
- `exportToPdf(schedule: ExamSchedule, viewType: ExportViewType, filePath: Path): void` – saves PDF (FR 21)
- `printSchedule(schedule: ExamSchedule, viewType: ExportViewType): void` – prints schedule (FR 21)

---

## Sequence Diagram Notes (`invigilator-sequence`)
Flow when the user clicks "Assign Invigilators" in the UI:
1. UI calls `InvigilatorAssigner.assignInvigilators(...)`.
2. `InvigilatorAssigner` loads the schedule, period settings (`ratio=40`, `minBreak=30`), and invigilators via `DataRepository`.
3. If there are no invigilators, it stops and returns without changing anything.
4. Otherwise it clears old assignments, loops over each exam and classroom allocation, checks eligibility rules (FR 44–47), picks the least-loaded invigilators (FR 48), records any slot shortages (FR 49), and attaches the invigilators to the allocation.
5. `DataRepository` saves the assignments to the local database (NFR 9), and the UI shows the result and any understaffed slot warnings[cite: 1, 2, 7, 8, 9, 11, 12, 13].
