# GUCera — A Stored-Procedure-Driven Educational Platform on ASP.NET Web Forms

GUCera is a three-role online learning platform — Student, Instructor, Admin — where every meaningful business rule lives inside the database. Course catalogues, prerequisite enforcement, weighted grade rollups, certificate issuance, promo codes, and authentication routing are all expressed as T-SQL stored procedures and executed by a thin ASP.NET Web Forms front end. The result is a textbook example of the "fat database, thin app" architecture: the C# code-behind is little more than a parameter shuttle between the page and SQL Server.

## Highlights

- **Three-role authentication with server-side routing.** A single `userLogin` stored procedure validates credentials and returns the user's role through `OUTPUT` parameters, which the application uses to dispatch to the Student, Instructor, or Admin landing page.
- **Prerequisite enforcement in SQL.** A student cannot enrol in a course until the database itself confirms every prerequisite has been completed — the check happens inside `enrollInCourse`, not in the application layer.
- **Weighted grade rollup in SQL.** `calculateFinalGrade` sums each assignment's weight × full-grade and the student's earned scores, then writes the final course grade back to the enrolment record in a single procedure.
- **Instructor course management.** Instructors create courses, define prerequisites, attach assignments with weights, grade submissions, view student feedback, and issue certificates — every action backed by a dedicated stored procedure.
- **Admin oversight tools.** Admins approve newly created courses, list users and instructors, view profiles, and issue promo codes through their own family of procedures.
- **52 server-side database objects** — 50+ stored procedures plus a scalar UDF — driving 28 Web Forms pages through ADO.NET parameterized commands.

## How It Works

The architecture is deliberately layered:

```
ASP.NET Web Forms page (.aspx)
        │
        ▼
C# code-behind (.aspx.cs)  ──  reads inputs, opens SqlConnection,
        │                       calls a stored procedure, binds the result
        ▼
Stored Procedure (T-SQL)   ──  validates state, enforces business rules,
        │                       returns rows or sets OUTPUT parameters
        ▼
SQL Server tables          ──  the system of record
```

The C# layer is intentionally thin: it opens a connection, builds a `SqlCommand` of `CommandType.StoredProcedure`, binds the user's inputs as `SqlParameter` values, and renders the result. Validation, authorization, eligibility checks, and aggregations all happen *inside* the database. This is the classic "fat database, thin app" pattern that the Databases I course is designed to teach — and it is what makes the project's SQL surface the most interesting part of the codebase.

## Featured Stored Procedures

A handful of procedures showcase the techniques the project leans on:

- **`userLogin`** — *OUTPUT parameters for role routing.* Verifies the credential pair, sets `@success` and `@type` (`0` Instructor, `1` Admin, `2` Student) so the application can pick the correct landing page in a single round-trip.
- **`enrollInCourse`** — *Business-rule enforcement at the relational level.* Confirms the instructor actually teaches the course, then checks `CoursePrerequisiteCourse` against the student's `StudentTakeCourse` history before inserting the enrolment row. The application never has to ask "can this student enrol?" — the database answers definitively.
- **`calculateFinalGrade`** — *Set-based weighted aggregation.* Computes the total weighted full-grade for a course's assignments and the student's matching earned scores in two `SUM` queries, then updates the enrolment row with the final grade. No loops in C#, no per-assignment round-trips.
- **`InstructorIssueCertificateToStudent`** — *Guarded inserts with `RAISERROR`.* Issues a course completion certificate only when the student's final grade clears the threshold; otherwise raises a SQL error the application surfaces to the user.
- **`checkStudentEnrolledInCourse`** *(scalar UDF)* — *Composable predicate.* Returns a `BIT` indicating enrolment, used inline by other procedures and by the application to gate access to course-content pages.

## Tech Stack

- **.NET Framework 4.7.2** — runtime target.
- **ASP.NET Web Forms** — page model and server controls.
- **C#** — code-behind for all 28 pages.
- **ADO.NET** — `SqlConnection`, `SqlCommand`, `SqlParameter`, `SqlDataReader`.
- **Microsoft SQL Server** — schema, stored procedures, the scalar UDF, and the data itself.

## Project Structure

```
GUCera/
├── GUCera.sln                       # Visual Studio solution
├── GUCera/                          # Web application project
│   ├── *.aspx + *.aspx.cs           # 28 Web Forms pages and code-behind
│   │     Login, Student / Instructor / Admin landings,
│   │     EnrollCourse, ListCourses, AddCourse, DefineAssignment,
│   │     SubmitAssignment, GradeSubmissions, ViewAssignmentGrades,
│   │     IssueCertificate, CreatePromoCodes, IssuePromoCode,
│   │     AddFeedback, ViewFeedback, ViewProfile, ...
│   ├── Web.config                   # ASP.NET configuration
│   └── GUCera.csproj                # project file
├── GUCera.sql                       # full schema + stored procedures
└── README.md
```

## Getting Started

You'll need **Visual Studio 2017 or newer** with the *ASP.NET and web development* workload, and a local **SQL Server** instance (Express, Developer, or LocalDB are all fine).

1. **Clone the repository**
   ```bash
   git clone https://github.com/anaselnemr/GUCera-Educational-Platform.git
   ```
2. **Create the database.** Open `GUCera.sql` in SQL Server Management Studio (or `sqlcmd`) and execute it against your SQL Server instance. The script creates the `GUCera` database, every table, and every stored procedure.
3. **Open the solution.** Launch `GUCera.sln` in Visual Studio.
4. **Configure the connection.** Update the SQL Server connection string in `Web.config` (and the matching value in the code-behind, if present) to point at your local instance.
5. **Run.** Press `F5`. Visual Studio will launch IIS Express and open the login page in your default browser.

## Course Context

Built for **Databases I (CSEN 504)** as part of the B.Sc. in Computer Science and Engineering at the **German University in Cairo (GUC)**, 2021. The brief was to design a non-trivial relational schema and implement the application's business logic *inside* the database — making it a natural showcase for stored-procedure design, OUTPUT parameters, set-based aggregation, and constraint-driven enforcement.

## Authors

Anas ElNemr  ·  Ahmed Eltawel
