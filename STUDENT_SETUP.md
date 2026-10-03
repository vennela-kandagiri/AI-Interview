# PyProctor AI — Student Setup & Run Guide

This guide describes the setup currently supported by this repository.

## Prerequisites

- Node.js 20.9 or newer (the current LTS release is recommended).
- Python 3.11 for the face-verification helper.
- Git, if you are cloning the repository rather than downloading it.
- A camera and microphone to test the interview experience.

Check your installations in a new terminal:

```text
node --version
python --version
git --version
```

Git is optional if you already have the project folder. On Windows, install
Python and Node.js using the official winget package IDs:

```powershell
winget install --id Git.Git -e --silent
winget install --id Python.Python.3.11 -e --silent
winget install --id OpenJS.NodeJS.LTS -e --silent
```

On macOS with Homebrew:

```bash
brew install git node python@3.11
```

Restart the terminal after installing tools so its PATH is refreshed.

## 1. Configure and install the backend

Open a terminal in the project root, then:

```text
cd backend
npm install
```

Create or edit `backend/.env`. There is no `.env.example` in this repository;
do not overwrite an existing `.env`, since it may contain local credentials.
The following is a local-development template—replace each placeholder:

```dotenv
DATABASE_URL="file:./dev.db"
PORT=5000
JWT_SECRET="replace-with-a-long-random-secret"
FRONTEND_URL="http://localhost:3000"

SMTP_HOST="smtp.gmail.com"
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER="your-email@gmail.com"
SMTP_PASS="your-gmail-app-password"
SMTP_FROM="PyProctor AI <your-email@gmail.com>"
```

`DATABASE_URL` points Prisma to a SQLite file at `backend/prisma/dev.db`.
The backend uses SQLite; a separate MySQL or PostgreSQL server is not required.

Email settings are required for the current invitation flow: the backend sends
an email when granting interview access. For Gmail, use an App Password rather
than your normal account password. Keep `.env` private and never commit or share
real credentials.

Generate the Prisma client and create/update the local SQLite tables:

```text
npx prisma generate
npx prisma db push
npm run seed
```

The backend also seeds any missing interview questions automatically when you
start it with `npm run dev`.

## 2. Set up the Python face-verification helper

This helper uses `face_recognition` and OpenCV. Installing `face_recognition`
also installs its `dlib` dependency, which may need native build tools if a
compatible prebuilt package is unavailable.

### Windows PowerShell

From the project root:

```powershell
cd face-test-python
python -m venv venv
.\venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install cmake
pip install -r requirements.txt
deactivate
cd ..
```

If PowerShell blocks virtual-environment activation, use Command Prompt and
activate with `venv\Scripts\activate.bat`, or follow your organization's
PowerShell execution-policy guidance. If installing `dlib` fails, install the
Visual C++ Build Tools and CMake, then retry the requirements installation.

### macOS or Linux

From the project root:

```bash
cd face-test-python
python3.11 -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip
pip install cmake
pip install -r requirements.txt
deactivate
cd ..
```

The backend attempts to locate this virtual environment when running face
matching. For reliable local development, activate it in the backend terminal
before starting the backend; that also puts its Python executable on `PATH`.
The helper can also be tested directly with a clear reference photo that
contains a face:

```text
cd face-test-python
```

On Windows, activate it with `.\venv\Scripts\Activate.ps1`; on macOS/Linux,
use `source venv/bin/activate`. Then run:

```text
python verify_face.py --known student.jpg
```

Replace `student.jpg` with the path to your own reference image. The script
opens the webcam; press Esc to exit.

## 3. Install the frontend

From the project root:

```text
cd frontend
npm install
cd ..
```

The frontend's development configuration forwards `/api` and `/uploads`
requests to the backend at `http://127.0.0.1:5000`; no additional proxy
configuration is needed for local development.

## 4. Run the applications

Start the backend in one terminal:

```text
cd backend
```

Activate the Python environment in that same terminal if you want backend face
matching to use its installed libraries. On Windows PowerShell:

```powershell
..\face-test-python\venv\Scripts\Activate.ps1
```

On macOS/Linux:

```bash
source ../face-test-python/venv/bin/activate
```

Then start the backend:

```text
npm run dev
```

Start the frontend in a second terminal:

```text
cd frontend
npm run dev
```

Open <http://localhost:3000> in your browser. Keep both terminals running.
The Python webcam test above is optional and is not a separate server.

## Open an interview invitation on a phone

The phone must use a public HTTPS URL rather than `localhost`. Start the
frontend and backend on the computer, then start an ngrok tunnel for the
frontend port:

```text
ngrok http 3000
```

Set `FRONTEND_URL` in `backend/.env` to the HTTPS forwarding URL printed by
ngrok, then restart the backend. New invitation emails will contain a link
that the candidate can open on a phone. Keep the ngrok tunnel and both app
servers running during the interview. A free ngrok URL can change when the
tunnel restarts, so update `FRONTEND_URL` and restart the backend if it changes.

## Technology and infrastructure notes

- Frontend: Next.js, React, and TypeScript. Browser-side vision dependencies
  include face-api.js, TensorFlow.js, and COCO-SSD.
- Backend: Express and TypeScript on Node.js; Prisma ORM with SQLite; JWT and
  bcrypt for authentication; Multer for uploads; Nodemailer for email.
- Face helper: Python with `face_recognition`, OpenCV, NumPy, and Pillow.
- `docker-compose.yml` describes a MySQL service, while its PostgreSQL section
  is commented out. Neither service is used by the current Prisma schema or
  required for the local SQLite setup above.
- The backend requires at least 10 Easy/Medium questions for the candidate's
  role (or 10 General questions as a fallback) before it starts an interview.

## Common issues

### Backend requests fail

- Confirm the backend is running on port 5000.
- Confirm `DATABASE_URL` is set in `backend/.env` and Prisma commands completed.
- If port 5000 is already in use, stop the other backend process before
  restarting this one.

### Face matching cannot find Python or its libraries

- Create and install the `face-test-python/venv` environment as described above.
- Check that `venv\Scripts\python.exe` exists on Windows or `venv/bin/python`
  exists on macOS/Linux.
- If `dlib` fails to build, install the platform's required native C++ build
  tools and CMake, then retry `pip install -r requirements.txt`.

### Invitations are not sent

- Confirm the SMTP host, port, user, password, and sender address in
  `backend/.env`.
- For Gmail, use an App Password. Check the backend terminal for the SMTP error.

### Camera or microphone access is blocked

- Use a modern browser and allow camera and microphone access when prompted.
- If access was previously denied, change the site's permissions in the browser.
