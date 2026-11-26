# Capacity Connect
### A Digital Capacity Building and Learning Management Portal

> **Theme:** Smart Education  **Idea:** A learning management portal with courses, assessments and progress tracking
> **Technology:** Python · Django 6.1 · MongoDB (through `django-mongodb-backend`)

---

## Contents
1. [Objective of the project](#1-objective-of-the-project)
2. [The problem we are solving](#2-the-problem-we-are-solving)
3. [Who uses the system (roles)](#3-who-uses-the-system-roles)
4. [Feature-by-feature explanation](#4-feature-by-feature-explanation)
5. [How the system works (architecture)](#5-how-the-system-works-architecture)
6. [Database design (MongoDB)](#6-database-design-mongodb)
7. [Division of work: 4 team members](#7-division-of-work-4-team-members)
8. [Run the project on your computer](#8-run-the-project-on-your-computer)
9. [Connect to MongoDB and view the database](#9-connect-to-mongodb-and-view-the-database)
10. [Deploy the website online](#10-deploy-the-website-online)
11. [Likely viva questions, with short answers](#11-likely-viva-questions-with-short-answers)
12. [Limitations and future scope](#12-limitations-and-future-scope)

---

## 1. Objective of the project

Sir/Ma'am, the objective of **Capacity Connect** is to give an organisation **one central web portal** for:

1. **Training.** Employees take structured online courses made of modules and lessons.
2. **Assessment.** Their understanding is tested with timed, auto-graded quizzes.
3. **Tracking.** Progress is recorded automatically, so learners, trainers and managers always know where everyone stands.
4. **Competency development.** Every course is linked to skills (competencies), so each employee gets a live skill profile.
5. **Knowledge sharing.** Employees can post articles, tips and questions so knowledge is not stuck with individuals.

In short: **learn → test → track → certify → share**, all in one place.

---

## 2. The problem we are solving

In many organisations, training is scattered:
- Materials sit in emails, shared drives and printed handouts.
- Attendance and test scores are kept in Excel sheets that nobody updates.
- Managers cannot easily answer *"Who has completed the mandatory cybersecurity training?"*
- Certificates are made by hand and are easy to fake.
- Experienced employees' knowledge is lost when they leave.

**Capacity Connect solves this** by putting courses, tests, progress, certificates and knowledge in a single database-backed website with role-based access.

---

## 3. Who uses the system (roles)

| Role | What they can do |
|---|---|
| **Learner** (employee) | Browse courses, enroll, study lessons, take assessments, earn certificates, view their competency profile, post in the Knowledge Hub, appear on the leaderboard |
| **Trainer** | Everything a learner can do, **plus** create and edit courses, modules, lessons and quizzes, see each learner's progress, view analytics, download reports, post announcements |
| **Administrator** | Everything a trainer can do, **plus** manage all users and their roles, see organisation-wide analytics, and use the Django admin console for every table |

The role is stored in the `role` field of our custom `User` model. Pages check it through two decorators we wrote, `trainer_required` and `admin_required` (in `accounts/decorators.py`). Anyone else who opens those pages gets a *403 Access denied*.

---

## 4. Feature-by-feature explanation

### 4.1 Registration, login and profiles
- New users register with name, employee ID, department and designation. New accounts are always **Learners**; only an admin can promote someone.
- Passwords are **hashed** by Django (PBKDF2), never stored as plain text.
- Each user has a profile showing their courses, certificates, points, level and badges.
- Users can edit their profile and change their password.
- The login page has **one-click demo buttons** to make presentations quick.

### 4.2 Course catalog
- Shows all **published** courses as cards (draft courses are hidden from learners).
- **Search** by keyword, **filter** by category, level (Beginner/Intermediate/Advanced) or *Mandatory*, and **sort** by newest, most popular, top rated or A–Z.
- Each card shows the enrolled count, average rating, duration and, if you're enrolled, **your progress bar**.

### 4.3 Course detail page
- Overview, *What you'll learn* (outcomes), full syllabus (modules → lessons, with durations), the assessment, instructor, **competencies developed** and **learner reviews** (1–5 stars).
- A progress ring shows how much of the course you have completed.
- **Enroll** with one click.

### 4.4 Learning player
- A two-column layout: the syllabus is on the left, the lesson on the right.
- Lessons support formatted text (headings, lists, bold, code, tip boxes), **embedded YouTube/Vimeo videos** and **downloadable attachments**.
- Buttons for *Previous*, *Mark complete & continue* and *Next*.
- **Resume:** the system remembers your last lesson (`Enrollment.last_lesson`) and opens it next time.

### 4.5 Assessments (quizzes)
- Multiple-choice questions, each worth some points.
- Trainers set a **pass mark** (e.g. 60%), a **time limit** and **maximum attempts**.
- A **countdown timer** auto-submits when time runs out; it turns red in the last minute.
- A tracker shows which questions you have answered.
- **Auto-grading** on the server: score = points earned ÷ total points × 100.
- **Answer review:** after submitting, you see the correct and wrong options and the trainer's explanation.
- The **My Assessments** page shows pending/passed quizzes, average score, pass rate and a score trend chart.

> **Why grade on the server?** If grading happened in JavaScript, a student could open the browser developer tools and see the answers. Correct answers never leave the server until after submission.

### 4.6 Progress tracking and course completion
- Progress % = (lessons completed + quizzes passed) ÷ (total lessons + total quizzes) × 100.
- A course is **complete** only when **all lessons are done and every quiz is passed**.
- Completion is checked automatically every time a lesson is completed or a quiz is passed (`check_course_completion()` in `courses/services.py`).

### 4.7 Certificates
- Issued **automatically** on completion, with a unique ID such as `CC-1A2B3C4D`.
- A printable certificate page, which can be saved as PDF with the browser's print option.
- A **public verification page**: anyone (for example HR at another company) can enter the ID to confirm it is genuine, without logging in.

### 4.8 Competency framework
- 10 competencies in 3 areas: **Technical** (Data Literacy, Cyber Hygiene, …), **Behavioural** (Leadership, Communication, …) and **Functional** (Project Planning, Regulatory Compliance).
- Each course is mapped to competencies (a many-to-many relationship).
- A learner's score in a competency is the average progress across the courses linked to it. That gives a level of **Not started → Awareness → Developing (25%+) → Proficient (60%+) → Expert (90%+)**.
- Shown with a **radar chart**.

### 4.9 Knowledge Hub
- Four post types: **Article, Quick Tip, Question, Resource**.
- Tags, search, filter by type, sort by latest or most viewed.
- **Likes** (they update instantly without reloading the page, using JavaScript `fetch`), **comments/answers** and a view counter.
- A sidebar shows popular tags and top contributors.

### 4.10 Gamification (points, levels, badges, leaderboard)
Points are **calculated from real activity**, not stored, so they can never get out of sync:

| Activity | Points |
|---|---|
| Lesson completed | +10 |
| Course completed | +100 |
| Assessment | best score ÷ 2 (max 50) |
| Knowledge post | +15 |
| Comment | +5 |

- **Levels:** Starter (0) → Explorer (150) → Achiever (400) → Expert (800) → Master (1500)
- **6 badges:** First Steps, Course Finisher, Quiz Ace, Knowledge Sharer, Certified Pro, Committed Learner
- **Leaderboard** with a top-3 podium, filterable by department.

### 4.11 Dashboard
- A greeting that changes with the time of day, your level and points.
- Stat cards: enrolled, completed, certificates and average score.
- **Continue learning** with progress bars, a **14-day activity chart**, **recommended courses**, an **alert for mandatory training** not yet done, achievements, announcements and recent quiz results.
- Trainers and admins also get a **Training overview** panel.

### 4.12 Trainer tools: Course Studio
- Create and edit courses (title, category, level, duration, outcomes, competencies, draft/published, mandatory).
- A **curriculum builder**: add, edit and delete modules and lessons, including file uploads.
- A **quiz builder** with Questions, Settings and Results tabs.
- A **learner progress table** for each course.

### 4.13 Analytics and reports
- KPIs: active learners, enrollments, completion rate, average score and pass rate.
- 4 charts: **enrollment trend** (6 months), **course performance**, **department participation** and **category split**.
- Tables for course breakdown, assessment performance and top learners.
- **Export CSV**: a full enrollment report that opens in Excel.
- Trainers see only *their* courses; admins see everything.

### 4.14 Administration
- **User management:** search and filter users, change roles from a dropdown.
- **Announcements** with priority (Info / Important / Urgent); a red dot on the bell shows when there's news.
- **Django admin console** for direct access to every table.

### 4.15 UI/UX
- A custom design system written in plain CSS, with no Bootstrap: colour tokens, cards, buttons and badges.
- **Light and dark mode** (remembered in the browser).
- **Fully responsive**: the sidebar becomes a slide-out menu on mobile.
- **Micro-interactions:** cards lift with a coloured glow on hover, buttons show a shine and a loading spinner, numbers count up, progress bars and rings fill in, sections fade in as you scroll, a top progress bar appears while pages load, there's a back-to-top button, and pressing `/` jumps to search.
- Toast notifications for success and error messages.
- Respects the *reduce motion* accessibility setting.
- Icons and charts are bundled locally, so the site **works offline** during a demo.

---

## 5. How the system works (architecture)

```
   Browser (HTML + CSS + JS)
          │  HTTP request  (e.g. GET /courses/)
          ▼
   Django URL dispatcher ──► View function (Python logic, permission checks)
                                   │
                                   ▼
                         Django ORM (models.py)
                                   │  django-mongodb-backend translates
                                   │  ORM queries into MongoDB queries
                                   ▼
                         MongoDB  (database: capacity_connect)
                                   │
          ┌────────────────────────┘
          ▼
   Django template (HTML) is filled with data ──► HTTP response back to browser
```

Django follows the **MVT** pattern (Model–View–Template):
- **Model:** the structure of the data (`models.py`)
- **View:** the logic that decides what to show (`views.py`)
- **Template:** the HTML page (`templates/`)

### Project apps (each Django "app" is one module of the system)

| App | Responsibility |
|---|---|
| `accounts` | Custom User model with roles, login/register, profiles, user management, permission decorators |
| `courses` | Categories, competencies, courses, modules, lessons, enrollments, progress, reviews, certificates, Course Studio, the demo-data command |
| `assessments` | Quizzes, questions, choices, attempts, grading, quiz builder |
| `knowledge` | Knowledge Hub posts, comments, likes |
| `core` | Landing page, dashboard, analytics, CSV export, leaderboard, announcements, search, gamification logic |
| `capacity_connect` | Project settings, main URLs, MongoDB configuration |

### What happens when a learner passes a quiz (example flow)
1. The browser POSTs the selected answers to `/assessments/<id>/take/`.
2. `quiz_take()` checks the learner is enrolled and has attempts left.
3. It compares each answer with the correct `Choice` **on the server**, then calculates the score.
4. It saves an `Attempt` document in MongoDB (with the answers stored as JSON).
5. If passed, it calls `check_course_completion()`. If every lesson is done too, it sets `completed_at` and creates a `Certificate`.
6. It redirects to the result page with a success message.

---

## 6. Database design (MongoDB)

We use **MongoDB**, a NoSQL **document database**. Each Django model becomes a **collection**, and each record is a **document** (similar to JSON). Every document gets a unique `_id` of type **ObjectId** (24 hex characters, e.g. `6afbf746f3f2ae0a5a628702`).

### Main collections

| Collection | Stores | Key fields |
|---|---|---|
| `accounts_user` | Users | username, password (hashed), role, department, designation |
| `courses_category` | Course categories | name, icon, colour |
| `courses_competency` | Skills framework | name, area |
| `courses_course` | Courses | title, slug, category_id, instructor_id, level, status, is_mandatory |
| `courses_course_competencies` | Course ↔ competency links (many-to-many) | course_id, competency_id |
| `courses_module` | Modules of a course | course_id, title, order |
| `courses_lesson` | Lessons | module_id, title, content, video_url, attachment |
| `courses_enrollment` | Who joined which course | user_id, course_id, enrolled_at, completed_at, last_lesson_id |
| `courses_lessonprogress` | Completed lessons | user_id, lesson_id, completed_at |
| `courses_review` | Ratings | course_id, user_id, rating, comment |
| `courses_certificate` | Certificates | user_id, course_id, code (unique) |
| `assessments_quiz` | Quizzes | course_id, pass_mark, time_limit_minutes, max_attempts |
| `assessments_question` / `assessments_choice` | Questions and options | quiz_id / question_id, is_correct |
| `assessments_attempt` | Quiz attempts | user_id, quiz_id, score, passed, answers (JSON) |
| `knowledge_post` / `knowledge_comment` | Knowledge Hub | author_id, kind, tags, body |
| `knowledge_post_likes` | Who liked which post | post_id, user_id |
| `core_announcement` | Announcements | title, priority, body |

**Relationships:** one Course has many Modules, one Module has many Lessons, and one Course has many Quizzes. Users and Courses are linked through **Enrollment**.

**Data integrity:** *unique constraints* stop a user enrolling twice in the same course, getting two certificates for one course, or reviewing a course twice.

---

## 7. Division of work: 4 team members

The work was divided by importance and complexity. **Person 1 (Team Lead)** handled the core architecture and the most critical logic that everything else depends on.

### Person 1: Team Lead · Backend architecture & core logic (~40%)
*The foundation: without this, nothing else works.*
- Project planning, folder structure and choice of technology (Django + MongoDB)
- **MongoDB integration:** settings, ObjectId primary keys, MongoDB-specific migrations, the `oid` URL converter
- **Database design:** all models and relationships (`courses/models.py`, `assessments/models.py`, `accounts/models.py`)
- **Custom User model with roles** and the permission decorators
- **Progress tracking engine** and **course-completion logic** (`courses/services.py`)
- **Assessment grading engine:** server-side scoring, attempt limits, timer handling (`assessments/views.py`)
- **Automatic certificates** and public verification
- **Analytics and CSV report export** (`core/views.py`)
- Deployment setup (production settings, static files, environment variables)

**Can answer:** how Django talks to MongoDB, why grading is on the server, how progress % is calculated, how a certificate is generated, how roles and permissions work.

### Person 2: Course management & learning experience (~25%)
- **Course catalog** with search, filters and sorting
- **Course detail page:** syllabus, outcomes, reviews, enrollment
- **Learning player:** lesson navigation, video embeds, attachments, resume
- **Course Studio** for trainers: create/edit courses, modules and lessons, file uploads
- **Quiz builder** UI (add questions, settings, results tabs)
- **Competency framework** page and the proficiency calculation

**Can answer:** how a trainer builds a course, how YouTube links become embedded videos, how competency levels are calculated, how search/filtering works.

### Person 3: UI/UX design & front-end (~20%)
- **Design system** in `static/css/app.css`: colours, typography, cards, buttons, badges, tables
- **Responsive layout:** sidebar, top bar, mobile menu
- **Dark mode** with saved preference
- **All page templates** (`templates/`) and reusable partials (course card, form fields)
- **Hover effects and animations:** card glow, button shine, count-up numbers, scroll reveal, progress animations, tooltips, loading spinners
- **Charts** with Chart.js (dashboard, analytics, competency radar, score trend)
- Quiz timer and answered-question tracker in JavaScript (`static/js/app.js`)

**Can answer:** how dark mode works, how the page adapts to mobile, how charts get data from Django (`json_script`), why there's no Bootstrap.

### Person 4: Community, engagement & testing (~15%)
- **Knowledge Hub:** posts, tags, comments, AJAX likes
- **Gamification:** points, levels, badges, leaderboard (`core/services.py`)
- **Announcements** module
- **Global search** (courses, posts, people)
- **Demo data generator:** `python manage.py seed_demo` with realistic courses and activity
- **Testing** of every page and workflow for all three roles, plus documentation (this README)

**Can answer:** how points are calculated, how a like updates without reloading, how the demo data is generated, how the project was tested.

---

## 8. Run the project on your computer

### Requirements
- **Python 3.12 or newer**
- **MongoDB Community Server** running on `localhost:27017` ([download](https://www.mongodb.com/try/download/community))

### Steps (Windows PowerShell)
```powershell
# 1. Open the project folder
cd "C:\Users\ADMIN\OneDrive\Desktop\Pyhton project"

# 2. (Recommended) create a virtual environment
python -m venv venv
.\venv\Scripts\activate

# 3. Install the packages
pip install -r requirements.txt

# 4. Create the collections in MongoDB
python manage.py migrate

# 5. Load demo users, courses, quizzes and activity
python manage.py seed_demo            # use --reset to wipe and reload

# 6. Start the server
python manage.py runserver
```
Open **http://127.0.0.1:8000** in your browser. Keep the terminal open; press `Ctrl + C` to stop.

### Demo logins
| Role | Username | Password |
|---|---|---|
| Administrator | `admin` | `admin123` |
| Trainer | `priya` or `rohan` | `trainer123` |
| Learner | `aarav`, `meera`, `kabir`, `ananya`, `vikram`, … | `learner123` |

**Admin console:** http://127.0.0.1:8000/admin/ (log in as `admin`)

### If something goes wrong
| Problem | Fix |
|---|---|
| *"This site can't be reached"* | The server isn't running. Run `python manage.py runserver` and keep the window open. |
| `ServerSelectionTimeoutError` | MongoDB isn't running. Open **Services** (Win + R → `services.msc`) → *MongoDB Server* → Start. |
| *"Demo data already exists"* | Run `python manage.py seed_demo --reset`. |
| Port 8000 busy | Run `python manage.py runserver 8080` and open port 8080. |

---

## 9. Connect to MongoDB and view the database

### Option A: MongoDB Compass (graphical, easiest)
MongoDB Compass is already installed on this computer (Start menu → *MongoDB Compass*).

1. Open **MongoDB Compass**.
2. Click **"+ Add new connection"**.
3. In the URI box, enter:
   ```
   mongodb://localhost:27017
   ```
4. Click **Connect**.
5. In the left panel, open the database **`capacity_connect`**.
6. Click any collection, e.g. **`accounts_user`**, **`courses_course`** or **`assessments_attempt`**, to see its documents.

**Useful things to show in Compass:**
- **Filter** box: `{ "role": "trainer" }` in `accounts_user` shows only trainers
- **Filter** in `assessments_attempt`: `{ "passed": true }` shows only passed attempts
- **Filter** in `courses_enrollment`: `{ "completed_at": { "$ne": null } }` shows completed enrollments
- The **Schema** tab shows the field types of a collection
- Do something live in the website (e.g. pass a quiz), click **Refresh** in Compass, and the new document appears. *This is a good demo for the supervisor.*

### Option B: mongosh (command line)
```powershell
mongosh
```
```javascript
use capacity_connect
show collections                                       // list all collections
db.accounts_user.find({}, {username: 1, role: 1})      // users and their roles
db.courses_course.countDocuments()                     // number of courses
db.assessments_attempt.find({passed: true}).limit(5)   // 5 passed attempts
db.courses_certificate.find().sort({issued_at: -1})    // latest certificates
```

> **Note:** passwords appear as long hashes like `pbkdf2_sha256$...`. That is expected and shows they are stored securely.

### Option C: Django admin
Log in as `admin` at **/admin/**. It shows the same data in friendly tables where you can add, edit or delete records.

---

## 10. Deploy the website online

We use **MongoDB Atlas** (a free cloud database) together with **Render** (free hosting for Python web apps).

### Step 1: Create a cloud database (MongoDB Atlas)
1. Sign up at **https://www.mongodb.com/cloud/atlas** and create a **free M0 cluster**.
2. **Database Access** → *Add New Database User* → create a username and password (save them).
3. **Network Access** → *Add IP Address* → **Allow access from anywhere** (`0.0.0.0/0`). Render's IP addresses change, so this is needed.
4. **Clusters** → *Connect* → *Drivers* → copy the connection string. It looks like:
   ```
   mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/
   ```
   Replace `<user>` and `<password>` with the database user from step 2.

### Step 2: Put the code on GitHub
```powershell
git init
git add .
git commit -m "Capacity Connect"
git branch -M main
git remote add origin https://github.com/<your-username>/capacity-connect.git
git push -u origin main
```

### Step 3: Create the web service on Render
1. Sign up at **https://render.com** with your GitHub account.
2. **New +** → **Web Service** → choose your `capacity-connect` repository.
3. Fill in:
   | Setting | Value |
   |---|---|
   | Runtime | Python 3 |
   | Build Command | `pip install -r requirements.txt && python manage.py collectstatic --noinput && python manage.py migrate && python manage.py seed_demo` |
   | Start Command | `gunicorn capacity_connect.wsgi:application` |
4. Under **Environment Variables**, add:
   | Key | Value |
   |---|---|
   | `MONGODB_URI` | your Atlas connection string from step 1 |
   | `MONGODB_NAME` | `capacity_connect` |
   | `DJANGO_DEBUG` | `0` |
   | `DJANGO_SECRET_KEY` | a long random string (generate one with `python -c "import secrets; print(secrets.token_urlsafe(50))"`) |
   | `DJANGO_ALLOWED_HOSTS` | `capacity-connect.onrender.com` (your Render domain) |
   | `DJANGO_CSRF_TRUSTED_ORIGINS` | `https://capacity-connect.onrender.com` |
   | `PYTHON_VERSION` | `3.13.0` |
5. Click **Create Web Service**. After a few minutes the site is live at `https://<your-app>.onrender.com`.

`seed_demo` in the build command only loads demo data the first time; later deploys skip it.

### Step 4: View the online database
In **MongoDB Compass**, connect with the **Atlas connection string** (instead of `localhost`) and open the `capacity_connect` database. It shows the live website's data.

### Deployment notes
- With `DJANGO_DEBUG=0`, static files (CSS/JS) are served by **WhiteNoise**, and secure cookies and HTTPS settings switch on automatically.
- Render's free plan sleeps after inactivity, so the first visit can take ~30 seconds.
- Uploaded lesson attachments are stored on the server's disk, which Render's free plan resets on each deploy. For permanent uploads, use cloud storage (e.g. AWS S3 or Cloudinary).
- **Change the demo passwords** before sharing the live link publicly.

---

## 11. Likely viva questions, with short answers

**Q: Why did you choose Django?**
A: It is a mature Python framework with built-in authentication, an admin panel, form validation, protection against CSRF/XSS/SQL injection, and a clean MVT structure. That let us focus on features instead of plumbing.

**Q: Django normally uses SQL databases. How did you use MongoDB?**
A: We used **`django-mongodb-backend`**, the official MongoDB backend for Django. It translates Django ORM queries into MongoDB queries, so we still write normal Django models. We had to use `ObjectIdAutoField` as the primary key and keep separate migrations for Django's built-in apps (`mongo_migrations/`).

**Q: What is the difference between SQL and MongoDB?**
A: SQL stores data in tables with fixed columns. MongoDB stores flexible JSON-like **documents** in **collections**. MongoDB scales easily and handles varied data well; for example, we store quiz answers as a JSON field inside each attempt.

**Q: What is an ObjectId?**
A: MongoDB's default unique ID: a 12-byte value shown as 24 hex characters. It contains a timestamp, so IDs are roughly ordered by creation time.

**Q: Was there anything Django can do that the MongoDB backend can't?**
A: Yes. `prefetch_related` on many-to-many fields isn't supported, so on the competency page we query the link collection (`courses_course_competencies`) directly. We also do most counting and averaging in Python, which keeps queries simple.

**Q: How do you stop a learner from cheating on a quiz?**
A: Correct answers are never sent to the browser before submission, and grading happens on the server. Attempt limits are checked on the server, and the start time is stored in the session.

**Q: What if a learner refreshes the page during a quiz?**
A: The page reloads with a fresh timer and nothing is submitted. Attempts are counted only on submission. The browser also warns "Your answers will be lost" before leaving.

**Q: How is progress percentage calculated?**
A: (completed lessons + passed quizzes) ÷ (total lessons + total quizzes) × 100.

**Q: When is a certificate generated?**
A: Automatically, the moment a learner has finished every lesson **and** passed every quiz of a course. `check_course_completion()` runs after each lesson completion and each passed quiz.

**Q: How can someone verify a certificate is real?**
A: Each certificate has a unique random code (generated with Python's `secrets` module). The public `/courses/certificates/verify/` page looks it up in the database.

**Q: How are passwords stored?**
A: Hashed with PBKDF2-SHA256 and a salt, Django's default. Even the admin cannot see the original password.

**Q: How do you stop a learner from opening trainer pages?**
A: The `@trainer_required` and `@admin_required` decorators check the user's role before the view runs and return 403 Forbidden otherwise. Trainers can also only edit **their own** courses (`can_manage_course()`).

**Q: What is CSRF and how did you handle it?**
A: Cross-Site Request Forgery is when another website tricks your browser into submitting a form. Every form in our app includes Django's `{% csrf_token %}`, and the server rejects requests without a valid token.

**Q: Why are points calculated instead of stored?**
A: So they can never get out of sync. If a post is deleted, the points disappear automatically, because they are always recomputed from real activity.

**Q: How does the Like button work without reloading the page?**
A: JavaScript sends a `fetch()` POST request. Django returns JSON (`{"liked": true, "count": 5}`) and the script updates the button.

**Q: How does dark mode work?**
A: All colours are CSS variables. Switching sets `data-theme="dark"` on the `<html>` tag, which swaps the variables. The choice is saved in `localStorage`.

**Q: How do charts get data from Django?**
A: The view prepares Python lists. The template outputs them safely with Django's `json_script` filter, and Chart.js reads that JSON to draw the chart.

**Q: How did you test the project?**
A: With an automated script using Django's test client. It opened all pages as admin, trainer and learner (117 checks), confirmed learners get 403 on trainer pages, and ran full workflows end to end: register → enroll → complete lessons → pass quiz → certificate; trainer creates course → module → lesson → quiz → question; post → like → comment.

**Q: What does `seed_demo` do?**
A: It is a custom Django management command that fills the database with realistic demo data (15 users, 8 courses, 31 lessons, 27 quiz questions, enrollments, attempts, certificates, posts), so the portal can be demonstrated immediately.

**Q: What is a slug?**
A: A URL-friendly version of a title, e.g. "Data Analytics Essentials" → `data-analytics-essentials`. It makes URLs readable: `/courses/data-analytics-essentials/`.

---

## 12. Limitations and future scope

**Current limitations**
- Questions are single-answer multiple choice only.
- No email notifications yet.
- Uploaded files are stored locally (not cloud storage).

**Future enhancements**
- More question types (multi-select, true/false, short answer)
- Email reminders for mandatory training deadlines
- Learning paths (a recommended sequence of courses per job role)
- Live virtual classroom integration (Zoom/Teams links with attendance)
- An AI assistant that recommends courses based on competency gaps
- A mobile app using the same backend through a REST API

---

## Project structure
```
Pyhton project/
├── manage.py                 Django command-line tool
├── requirements.txt          Python packages
├── capacity_connect/         Settings, main URLs, MongoDB app configs, ObjectId URL converter
├── accounts/                 Users, roles, login/register, profiles, user management
├── courses/                  Courses, lessons, progress, certificates, competencies, studio
│   ├── services.py           Progress & completion engine
│   └── management/commands/  seed_demo command and demo content
├── assessments/              Quizzes, grading, attempts, quiz builder
├── knowledge/                Knowledge Hub
├── core/                     Dashboard, analytics, leaderboard, announcements, search
│   ├── services.py           Points, levels, badges
│   └── templatetags/         Markdown formatter, nav highlighting, helpers
├── templates/                HTML pages
├── static/                   CSS design system, JavaScript, bundled Chart.js and Lucide icons
└── mongo_migrations/         MongoDB migrations for Django's built-in apps
```
