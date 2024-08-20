## Challenge Overview: Flask-Based Competitive Programming Web Application
In this challenge, we're presented with a Flask-based Competitive Programming web application, and we're provided with the application's code and an SQLite database. The main objective is to extract a hidden flag that is stored as the expected output of a test case for a specific problem.

## Code Analysis and Database Schema
The application consists of three main files:

app.py: The main Flask application handling submissions and evaluation.
db.py: Handles the database connections and inserts the flag into the database.
sandbox.py: Contains the sandboxing code that limits the actions a submission can perform.
From db.py, we observe that the flag is stored as the output for a hidden test case associated with the problem helloinput:

```python
Copy code
with Session(engine) as db:
    flag = os.environ.get("FLAG")
    if flag:
        flag_case = db.scalar(select(ProblemTestCase).filter_by(problem_id="helloinput", hidden=True))
        flag_case.output = flag + "\n"
        db.commit()
```
The test cases for each problem are stored in the problem_test_cases table in db.sqlite. The table schema is as follows:

```sql
CREATE TABLE IF NOT EXISTS "problem_test_cases" (
    "id" INTEGER NOT NULL,
    "problem_id" VARCHAR,
    "input" VARCHAR,
    "output" VARCHAR,
    "hidden" INTEGER,
    FOREIGN KEY("problem_id") REFERENCES "problems"("id"),
    PRIMARY KEY("id")
);
```
Our target is to retrieve the output of the hidden test case for helloinput, which contains the flag.

## Understanding the Submission Handling Process
In app.py, the submission handling logic involves:

Storing the submission in the submissions table.
Retrieving the test cases from the problem_test_cases table.
Writing the test case input and expected output to temporary files (/tmp/<submission_id>.in and /tmp/<submission_id>.expected).
Executing the submitted code in a sandboxed environment.
Comparing the actual output with the expected output using the diff command.
Sandbox Bypassing via Race Condition
The sandboxing code in sandbox.py restricts operations such as opening files in write mode and reading the .expected file associated with the current submission ID. However, the .expected file is temporarily created during the submission process, providing a window of opportunity to read it before it gets deleted.

## Exploiting the Race Condition
To exploit this, we can use a race condition between two submissions:

Submission A: A dummy submission that prints the expected output for the first test case and enters an infinite loop for the second test case. This delay prevents the .expected file from being deleted immediately.

### Code for Submission A:

```python
if input() == "Welcome to Crator":
    print("Welcome to Crator")
else:
    while True:
        pass
```
Submission B: A submission that repeatedly attempts to read the .expected file created by Submission A.

### Code for Submission B:

```python
print(open("/tmp/1.expected").read())
```
## Execution Flow
- Start Submission A, which will hang during the second test case, keeping the /tmp/<submission_id>.expected file intact.
- While Submission A is still running, submit Submission B, which tries to read the .expected file created by Submission A.
