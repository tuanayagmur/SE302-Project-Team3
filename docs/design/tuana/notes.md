# Tuana – design notes (T2)

Files: `architecture.drawio`, `architecture.png`, `class-diagram.drawio` (pages *Model* and *Logic*), `class-diagram-model.png`, `class-diagram-logic.png`

## Architecture (`architecture.png`)

Exam Scheduler is a desktop application that runs on one Windows computer and does not connect to any other system. It reads the input files (courses, students, classrooms, attendance lists, exam durations and the optional invigilator list), keeps its data in a local SQLite database, and writes the schedule to CSV and PDF files.

We chose a three-tier architecture and used MVC in the upper part:
- Presentation layer: `view` (JavaFX screens) and `controller` (one controller per screen, it gets the user actions and calls the logic classes).
- Application logic layer: `model` (data classes) and `logic` (import, scheduling, checking, updating, invigilators, export).
- Storage layer: `storage` (`DataRepository`), the only package that uses the database.

Each layer only uses the layer below it. This way we can change the screens without changing the scheduling code, and the logic can be tested without the user interface. The screens observe the model (Observer pattern), so when the data or the schedule changes, for example when the schedule becomes outdated, the screens are updated.

## Model classes (`class-diagram-model.png`)

A `Student` takes many courses and a `Course` has many students (FR 1, 2). A course has an exam duration, a difficulty level and the classroom features it needs, and it can be fixed to one `TimeSlot` (FR 5, 9, 12, 13). `ExamPeriod` contains the time slots and also keeps the settings like changeover time, default exam duration and the invigilator settings (FR 4, 36).

The result is an `ExamSchedule` that has one `ScheduledExam` for each course (FR 7, 28). Each exam is in one time slot and in one or more `ClassroomAllocation`s, so a big exam can be split into several classrooms and each student knows their classroom (FR 33, 34). Invigilators are optional (0..*), so the schedule can be created without an invigilator list (FR 14–16). The schedule has a status, `VALID` or `OUTDATED`, and `markOutdated()` is called when the data changes (FR 26).

## Logic classes (`class-diagram-logic.png`)

`DataImporter` reads and checks one input file and returns the errors; the data is saved only after the user confirms (FR 1, 3, 6, 23, 24). `DataRepository` saves and loads the data, the exam period and the schedule (FR 27). `ScheduleGenerator` first checks if a schedule is possible and then creates it, and `ScheduleValidator` checks the rules, also when the user moves an exam by hand (FR 7, 8, 28–40). `ScheduleUpdater` updates only the exams affected by a data change (FR 26, 42). `InvigilatorAssigner` runs only if there is an invigilator list and returns warnings when there are not enough invigilators (FR 16, 43–49). `ScheduleExporter` exports and prints the schedule (FR 20, 21, 54). The algorithm is explained in `algorithm.md`.

## Changes from the first class list
- Added `ScheduleStatus` and `FileType` enumerations.
- Added `loadExamPeriod()` / `saveExamPeriod()` to `DataRepository`, because the exam period is entered in the app, not imported.
- `DataImporter` does not call `ScheduleUpdater`; the controller calls it after saving.
