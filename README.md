# GUCera — ASP.NET Educational Platform with Stored-Procedure-Driven Backend

GUCera is an online-learning marketplace built as the term project for **Databases I (CSEN 504)** at the **German University in Cairo (GUC)**, B.Sc. Computer Science & Engineering, ~2021. It is a three-role ASP.NET Web Forms application — Student, Instructor, Admin — sitting on top of a SQL Server schema where **every database mutation is funnelled through a stored procedure**, never through ad-hoc SQL composed in C#. The point of the project (and this codebase) is to demonstrate end-to-end DDL / DML / parameterized stored procedures / a scalar UDF wired into a real .NET front-end via `SqlCommand` + `CommandType.StoredProcedure` with typed `SqlParameter`s.

---

## Features

What the platform actually does, by the three role surfaces shipped in the repo:

### Authentication & registration
- Single login page (`Login.aspx`) that calls one stored procedure (`userLogin`) with two `OUTPUT` parameters — `@success bit` and `@type int` — and routes to `Student.aspx`, `Instructor.aspx`, or `Admin.aspx` based on the role bit (0 = instructor, 1 = admin, 2 = student).
- Self-service registration for both roles (`Student Registration.aspx`, `Instructor Registration.aspx`) → `studentRegister` / `InstructorRegister` SPs that insert into `Users` and use `SCOPE_IDENTITY()` to chain into the role-specific child table in one transaction.
- Optional secondary phone number on the profile (`AddMobile.aspx` → `addMobile` SP, multi-valued via the `UserMobileNumber` join table).

### Student surface (`Student.aspx` and friends)
- Browse the catalogue of accepted courses (`ListAllCourses.aspx`, `ListCourses.aspx` → `availableCourses` SP that LEFT-OUTER-JOINs `StudentTakeCourse` to filter out courses the student is already enrolled in).
- Enroll in a course with a chosen instructor (`EnrollCourse.aspx` → `enrollInCourse` SP, which validates that the instructor actually teaches the course AND that all prerequisites in `CoursePrerequisiteCourse` have been completed before allowing the insert).
- Add credit cards (`AddCreditCard.aspx` → `addCreditCard` SP, two-row insert into `CreditCard` + `StudentAddCreditCard`).
- Redeem promo codes issued by admins (`ViewPromo.aspx` → `viewPromocode`, `ExistMultiPromo`).
- View / submit assignments (`ViewAssignments.aspx` + `SubmitAssignment.aspx` → `viewAssign`, `submitAssign`).
- See per-assignment grades and the computed final grade (`ViewAssignmentGrades.aspx` → `viewAssignGrades` and `viewFinalGrade`, both with `OUTPUT` decimal parameters).
- Leave course feedback and rate instructors (`AddFeedback.aspx` → `addFeedback`, `rateInstructor` — both gated by the `checkStudentEnrolledInCourse` UDF).
- Pull issued certificates (`ListCertificates.aspx`, `ViewProfile.aspx` → `viewCertificate`).
- View / edit own profile (`ViewProfile.aspx` → `viewMyProfile`, `editMyProfile` with a column-by-column `IF (@param IS NOT NULL)` update pattern).

### Instructor surface (`Instructor.aspx`)
- Add a new course awaiting admin approval (`AddCourse.aspx` → `InstAddCourse`, which inserts into `Course` with `accepted = NULL` and simultaneously links the creator into `InstructorTeachCourse`).
- Co-teach: invite another instructor onto your own course (`AddAnotherInstructorToCourse` SP).
- Update content / description on accepted courses (`UpdateCourseContent`, `UpdateCourseDescription` — both ownership-gated by `instructorId = @instrId`).
- Define course prerequisites (`DefineCoursePrerequisites` SP).
- Define assignments — quiz, exam, or project, with weight + deadline + content (`DefineAssignment.aspx` → `DefineAssignmentOfCourseOfCertianType`; type validated by the `CHECK (type in ('quiz', 'exam', 'project'))` constraint on `Assignment`).
- View student submissions (`ViewSubmissions.aspx` → `InstructorViewAssignmentsStudents`).
- Grade submissions (`GradeSubmissions.aspx` → `InstructorgradeAssignmentOfAStudent`).
- Compute and persist the final course grade as the weight-summed sum of assignment grades (`calculateFinalGrade` SP — see *Most interesting SQL artefact* below).
- Issue a course-completion certificate to a student who passed (`IssueCertificate.aspx` → `InstructorIssueCertificateToStudent`, gated by `grade > 2.0`).
- Read student feedback on own course (`ViewFeedback.aspx` → `ViewFeedbacksAddedByStudentsOnMyCourse`).

### Admin surface (`Admin.aspx`)
- List every instructor and student in the system (`AdminListInstr`, `AdminListAllStudents`).
- Drill into any user's profile (`AdminViewInstructorProfile`, `AdminViewStudentProfile`).
- Browse all courses in the catalogue (`ListAllCourses.aspx` → `AdminViewAllCourses`).
- Triage instructor-submitted courses (`PendingCourses.aspx` + `AcceptPendingCourses.aspx` → `AdminViewNonAcceptedCourses` and `AdminAcceptRejectCourse`, the latter stamping the approving admin's id onto the course).
- Mint promo codes (`CreatePromoCodes.aspx` → `AdminCreatePromocode`, with full null-guard validation).
- Issue a promo code to a specific student (`IssuePromoCode.aspx` → `AdminIssuePromocodeToStudent`).

---

## Tech Stack

- **Runtime**: .NET Framework **4.7.2**
- **Web framework**: ASP.NET **Web Forms** (`.aspx` pages with `code-behind` `.aspx.cs` files; auto-generated `.aspx.designer.cs` partials)
- **Language**: C# 7-era, `System.Web.UI.Page` base
- **Database**: **Microsoft SQL Server** (the dev `Web.config` ships with `Server=DESKTOP-N6HA9K1\SQLEXPRESS; Database=GUCera; Trusted_Connection=Yes`, i.e. SQL Server Express + Windows Auth)
- **Data access**: raw **ADO.NET** — `System.Data.SqlClient` (`SqlConnection`, `SqlCommand`, `SqlParameter`). No EF, no Dapper, no ORM.
- **Compiler**: Roslyn via `Microsoft.CodeDom.Providers.DotNetCompilerPlatform 2.0.1`
- **Container scaffolding**: `Microsoft.VisualStudio.Azure.Containers.Tools.Targets 1.10.9` (Docker launch profile present but no `Dockerfile` checked in)
- **Hosting target**: IIS Express (HTTPS on port `44390`) for development; project type GUID identifies it as an ASP.NET Web Application
- **Styling**: hand-rolled `default.css` + `fonts.css`, no front-end framework

---

## Database

The schema is defined in `GUCera.sql` (also mirrored at `GUCera/GUCera/GUCera.sql`). It models a small Coursera-style marketplace.

### Entities (18 tables)

| Table | Purpose |
|---|---|
| `Users` | Base identity row — `id` IDENTITY PK, `firstName`, `lastName`, `password` (plaintext — see *Honest caveats*), `gender` bit, `email` UNIQUE, `address` |
| `Student` / `Instructor` / `Admin` | Role tables — share PK with `Users` via `id int PRIMARY KEY REFERENCES Users(id)` to model 1:1 inheritance. `Student` carries `gpa`, `Instructor` carries `rating` |
| `UserMobileNumber` | Multi-valued attribute — `(id, mobileNumber)` composite PK |
| `Course` | `id` IDENTITY PK, `creditHours`, `name` UNIQUE, `courseDescription`, `price`, `content`, `adminId` (approver), `instructorId` (creator), `accepted` bit |
| `Assignment` | Weak-entity child of `Course` — composite PK `(cid, number, type)`, with `CHECK (type in ('quiz','exam','project'))` and `CHECK (weight between 0 and 100)` |
| `Feedback` | Per-course comments by a student — `(cid, number)` PK with `number int IDENTITY` (per-course identity) |
| `Promocode` | `code varchar(6)` PK, issue/expiry dates, `discount`, the issuing `adminId` |
| `CreditCard` | `number varchar(15)` PK, `cardHolderName`, `expiryDate`, `cvv` |
| `CoursePrerequisiteCourse` | Self-referential M:N on `Course` |
| `InstructorTeachCourse` | M:N — multiple instructors can co-teach one course |
| `StudentTakeCourse` | M:N enrollment — `(sid, cid, insid)` PK so a student-course pair is scoped to which instructor section they enrolled in. Carries `payedfor` and `grade` |
| `StudentTakeAssignment` | M:N submission — `(sid, cid, assignmentNumber, assignmenttype)` composite PK with a 4-column FK back into `Assignment` |
| `StudentRateInstructor` | M:N rating — `(sid, insid)` PK, drives `Instructor.rating` recompute |
| `StudentCertifyCourse` | M:N issued certificates with `issueDate` |
| `StudentAddCreditCard` | M:N |
| `StudentHasPromocode` | M:N |

Referential-integrity policy is intentional and varied: ownership chains use `ON DELETE CASCADE ON UPDATE CASCADE` (e.g. `Student → Users`); ledger-style tables use `ON DELETE NO ACTION` to preserve historical rows (e.g. `StudentCertifyCourse → Course`); `Promocode.adminId` uses `ON DELETE SET NULL` so promo codes survive admin deletion.

### Stored procedures (52 total) and 1 scalar UDF

The repo's whole DB philosophy is **"no ad-hoc SQL in C#"** — every code-behind opens a `SqlConnection`, configures `cmd.CommandType = CommandType.StoredProcedure`, binds `SqlParameter`s by name, and `ExecuteNonQuery` / `ExecuteReader`. Output values come back through `ParameterDirection.Output` parameters, exemplified by `userLogin` returning both `@success` and `@type` for the C# router to switch on.

**Auth & registration**
- `userLogin (@id, @password, @success OUTPUT, @type OUTPUT)` — single SP that authenticates AND classifies role in one round-trip
- `studentRegister`, `InstructorRegister` — two-table inserts with `SCOPE_IDENTITY()` chaining
- `addMobile` — multi-valued attribute insert

**Admin SPs**
`AdminListInstr`, `AdminViewInstructorProfile`, `AdminViewAllCourses`, `AdminViewNonAcceptedCourses`, `AdminViewCourseDetails`, `AdminAcceptRejectCourse`, `AdminCreatePromocode`, `AdminListAllStudents`, `AdminViewStudentProfile`, `AdminIssuePromocodeToStudent`

**Instructor SPs**
`InstAddCourse`, `UpdateCourseContent`, `UpdateCourseDescription`, `AddAnotherInstructorToCourse`, `InstructorViewAcceptedCoursesByAdmin`, `DefineCoursePrerequisites`, `DefineAssignmentOfCourseOfCertianType`, `ViewInstructorProfile`, `InstructorViewAssignmentsStudents`, `InstructorgradeAssignmentOfAStudent`, `ViewFeedbacksAddedByStudentsOnMyCourse`, `updateInstructorRate`, `calculateFinalGrade`, `InstructorIssueCertificateToStudent`

**Student SPs**
`viewMyProfile`, `editMyProfile`, `availableCourses`, `courseInformation`, `enrollInCourse`, `addCreditCard`, `viewPromocode`, `enrollInCourseViewContent`, `viewAssign`, `submitAssign`, `viewAssignGrades` (OUTPUT), `addFeedback`, `rateInstructor`, `viewCertificate`, `viewFinalGrade` (OUTPUT), `payCourse`

**Helper / utility SPs**
`ExistPromoCode`, `ExistStudent`, `ShowAllPromo`, `ExistMultiPromo`, `getid`, `getCourseID`

**Scalar UDF**
- `checkStudentEnrolledInCourse(@sid, @cid) RETURNS BIT` — used as a guard inside `addFeedback`, `rateInstructor`, `viewCertificate`, and `viewFinalGrade` so the gating logic is centralised in one place rather than re-spelled on every SP.

The schema does **not** define any SQL views — the project leans on parameterized SPs with output parameters and `INNER JOIN`s rather than persisted views.

### Most interesting SQL artefact

`calculateFinalGrade` does the weighted-rollup-and-persist that ties the whole grading system together:

```sql
CREATE PROC calculateFinalGrade @cid int, @sid int, @insId int AS
BEGIN
  declare @fullGrade int
  select @fullGrade = Sum((weight/100) * fullgrade) from Assignment where cid=@cid

  declare @studentScore int
  select @studentScore = sum(grade) from StudentTakeAssignment where sid=@sid and cid=@cid

  update StudentTakeCourse set grade = @studentScore
  where cid=@cid and sid=@sid and insid=@insId
END
```

It computes the weighted denominator from `Assignment.weight × Assignment.fullGrade`, sums the student's earned `StudentTakeAssignment.grade` rows, and writes the result back into the enrollment row keyed by `(sid, cid, insid)`. Honourable mention: `enrollInCourse`, which uses nested `IF EXISTS` to enforce the prerequisite rule (`cid in (select preid from CoursePrerequisiteCourse where cid=@cid)`) before allowing the insert.

---

## Project Structure

```
.
├── GUCera.sql                          ← root copy of the full DB script (DDL + 52 SPs + 1 UDF)
├── README.md
├── ReadMe.pdf                          ← original course-deliverable PDF
└── GUCera/
    ├── GUCera.sln                      ← Visual Studio 2019 solution (Format 12, VS v16)
    └── GUCera/                         ← ASP.NET Web Application project
        ├── GUCera.csproj               ← .NET Framework 4.7.2 web app
        ├── GUCera.sql                  ← in-project copy of the DB script
        ├── Web.config                  ← connection string + Roslyn compiler config
        ├── Web.Debug.config / Web.Release.config
        ├── packages.config             ← NuGet packages
        ├── default.css / fonts.css     ← hand-rolled styling
        ├── Login.aspx (+ .cs + .designer.cs)
        ├── Student Registration.aspx   ← role-specific signup
        ├── Instructor Registration.aspx
        │
        ├── Student.aspx                ← landing/dashboard for the Student role
        ├── ListCourses.aspx            ← browse the catalogue
        ├── ListAllCourses.aspx
        ├── EnrollCourse.aspx
        ├── ViewAssignments.aspx
        ├── SubmitAssignment.aspx
        ├── ViewAssignmentGrades.aspx
        ├── AddFeedback.aspx
        ├── AddCreditCard.aspx
        ├── ViewPromo.aspx
        ├── ListCertificates.aspx
        ├── ViewProfile.aspx
        ├── AddMobile.aspx
        │
        ├── Instructor.aspx             ← landing for the Instructor role
        ├── AddCourse.aspx
        ├── DefineAssignment.aspx
        ├── ViewSubmissions.aspx
        ├── GradeSubmissions.aspx
        ├── IssueCertificate.aspx
        ├── ViewFeedback.aspx
        │
        ├── Admin.aspx                  ← landing for the Admin role
        ├── PendingCourses.aspx
        ├── AcceptPendingCourses.aspx
        ├── CreatePromoCodes.aspx
        ├── IssuePromoCode.aspx
        │
        └── bin/, .vs/, packages/       ← build output + Visual Studio metadata
```

Every page comes as a triplet — the markup `.aspx`, the C# code-behind `.aspx.cs`, and the auto-generated `.aspx.designer.cs` partial that wires up server controls.

---

## How to Build & Run

### Prerequisites

- **Visual Studio 2019 or later** (the solution targets VS 16) with the **ASP.NET and web development** workload installed
- **.NET Framework 4.7.2** Developer Pack
- **SQL Server** (Express edition is fine — that's what the dev config targets)
- **SQL Server Management Studio (SSMS)** or any T-SQL client capable of running the `GUCera.sql` script

### Steps

1. **Clone the repository**
   ```bash
   git clone https://github.com/anaselnemr/GUCera-Educational-Platform.git
   cd GUCera-Educational-Platform
   ```

2. **Provision the database**
   - Open SSMS and connect to your SQL Server instance.
   - Open `GUCera.sql` (root) and execute the whole script. It will:
     - `CREATE DATABASE GUCera;`
     - `USE GUCera;`
     - Create the 18 tables, 1 scalar UDF, and 52 stored procedures.

3. **Point the web app at your SQL Server**
   - Open `GUCera/GUCera/Web.config` and edit the connection string (the checked-in value `Server=DESKTOP-N6HA9K1\SQLEXPRESS;` is the original developer's machine name and will not resolve on yours):
     ```xml
     <connectionStrings>
       <add name="GUCera"
            connectionString="Server=YOUR_SERVER\SQLEXPRESS;Database=GUCera;Trusted_Connection=Yes;"/>
     </connectionStrings>
     ```

4. **Open + build the solution**
   - Open `GUCera/GUCera.sln` in Visual Studio.
   - Right-click the solution → *Restore NuGet Packages*.
   - `Build → Build Solution` (Ctrl+Shift+B).

5. **Run the app**
   - Press F5 (Debug) or Ctrl+F5 (Run without Debugging). The project is configured to launch via **IIS Express** on `https://localhost:44390/Login.aspx`.
   - Register a Student or Instructor from the login screen, or — to test the Admin flow — manually `INSERT` a row into `Users` and a matching row into `Admin` via SSMS, since the UI does not expose admin self-registration.

---

## Coursework Context

This repository is the term project for **Databases I (CSEN 504)** at the **German University in Cairo (GUC)**, B.Sc. Computer Science & Engineering, taken by the authors in approximately **2021**. The course's deliverable spec asked teams to build an end-to-end multi-role web application demonstrating mastery of:

- **DDL** — schema design, primary/foreign keys, composite keys, weak-entity modelling, `CHECK` constraints, varied `ON DELETE`/`ON UPDATE` referential actions
- **DML** — non-trivial multi-table queries, joins (`INNER`, `LEFT OUTER`), aggregates (`SUM`, `AVG`), correlated subqueries
- **Stored procedures** — input/output parameters, error handling with `RAISERROR`, control flow, transactions via `SCOPE_IDENTITY()`
- **User-defined functions** — scalar UDF reused as a centralised guard
- **Application integration** — calling all of the above from a real client (here: ASP.NET Web Forms + ADO.NET)

GUCera satisfies that brief end-to-end. The full original write-up is in `ReadMe.pdf`.

---

## Authors

This repository is a personal mirror by **Anas ElNemr** of the joint coursework, kept for portfolio and reference. The original team for the Databases I deliverable was four contributors:

- **Anas ElNemr** — co-author ([@anaselnemr](https://github.com/anaselnemr))
- **Ahmed Eltawel** — co-author ([@ahmedeltawel](https://github.com/ahmedeltawel))
- **Mahmoud Nabil**
- **Ahmed Farouk**

All four share equal credit for the academic submission.

---

## Acknowledgements

Friendly mirror of, and full credit shared with, the canonical upstream repository at **[github.com/ahmedeltawel/DB1-SQL-GUCera](https://github.com/ahmedeltawel/DB1-SQL-GUCera)** maintained by Ahmed Eltawel. This fork exists for portfolio purposes; both repositories represent the same joint course project and both authors share full ownership.

Thanks to the **GUC CSEN 504 (Databases I)** teaching team, ~2021 academic year, for the project specification.

---

## Honest caveats

In the spirit of "list what's actually there":

- **Passwords are stored in plaintext** in `Users.password varchar(20)`. Course-project scope, never deployed publicly — but worth flagging if you ever want to extend this.
- The connection string in `Web.config` is **machine-pinned to the original developer's host** (`DESKTOP-N6HA9K1\SQLEXPRESS`) and **not parameterised**.
- A handful of small issues survive in the SQL: `payCourse` references `studenStudentTakeCourse` (typo for `StudentTakeCourse`); `ExistPromoCode` calls `count()` instead of `count(*)`; `editMyProfile` declares `@email varchar(10)` but `Users.email` is `varchar(50)`.
- There are **no SQL views** — the design favours parameterized stored procedures with `OUTPUT` parameters.
- There is **no automated test suite, no CI, no migration framework** — `GUCera.sql` is run once by hand against a fresh database.
- The repo includes built `bin/` artefacts, the original `.vs/` folder, and a `.DS_Store` that was committed accidentally on a macOS development pass.
