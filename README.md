# 📋 Attendly – Employee Attendance System

A lightweight, self-contained **employee attendance web application** built with vanilla JavaScript and Supabase. Employees log in using an **organization code** and a **4‑digit PIN**, capture a **selfie** via their webcam, and record **punch in / punch out** with work reason tracking. Monthly reports with calendar views help employees and employers monitor attendance.

## ✨ Features

- **🔐 Secure PIN Login** – Employees authenticate using an organization code + 4‑digit PIN (hashed with SHA‑256).
- **📸 Selfie Capture** – Automatic photo capture via webcam during punch in/out. Images are uploaded to a Supabase storage bucket.
- **⏱ One‑Tap Punch In/Out** – Simple toggle button with visual status feedback.
- **📝 Work Reason Tracking** – When punching out, employees choose a reason:
  - **Office / Work** → must specify a sub‑reason (e.g., client meeting, field work, training).
  - **Personal** → no extra details needed.
- **📊 Monthly Report** – Calendar view showing daily attendance status (Present / Absent / Half‑Day) with detailed punch times and reasons on each day.
- **🔧 Built‑in Settings** – Password‑protected Supabase configuration screen (URL, anon key, service role key) – useful for developers or admins.
- **🔒 Row Level Security (RLS)** – Pre‑defined RLS policies ensure employers can only access their own organizations and employees.
- **💾 Local Storage** – Remembers the last used organization code and employee session for a smoother login flow.

---

## 🛠 Tech Stack

- **Frontend** – Vanilla HTML / CSS / JavaScript (single HTML file)
- **Backend** – [Supabase](https://supabase.com/) (PostgreSQL + Auth + Storage)
- **Authentication** – SHA‑256 PIN hashing (client‑side)
- **Charts** – [Chart.js](https://www.chartjs.org/) (for future extensibility – already included)
- **Deployment** – Any static hosting (Netlify, Vercel, GitHub Pages, or even local file)

---

## 👨‍💻 Developer Setup

### 1. Supabase Project

1. Create a new project on [Supabase](https://supabase.com/).
2. In the **SQL Editor**, run the following schema script (includes tables, RLS, and policies):

```sql
-- Organizations
CREATE TABLE organizations (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  employer_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  org_code TEXT NOT NULL,
  org_name TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Employees
CREATE TABLE employees (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  employee_id TEXT NOT NULL,
  name TEXT NOT NULL,
  role TEXT,
  password_hash TEXT NOT NULL,
  pin_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(organization_id, employee_id)
);

-- Roles (for autocomplete)
CREATE TABLE roles (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  role_name TEXT NOT NULL,
  UNIQUE(organization_id, role_name)
);

-- Attendance
CREATE TABLE attendance (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  employee_id UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  punch_in_time TIMESTAMPTZ NOT NULL,
  punch_out_time TIMESTAMPTZ,
  date DATE NOT NULL,
  punch_in_image_url TEXT,
  punch_out_image_url TEXT,
  punch_out_reason TEXT,            -- 'office' or 'personal'
  punch_out_reason_detail TEXT,     -- optional sub‑reason
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Enable RLS
ALTER TABLE organizations ENABLE ROW LEVEL SECURITY;
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;
ALTER TABLE roles ENABLE ROW LEVEL SECURITY;
ALTER TABLE attendance ENABLE ROW LEVEL SECURITY;

-- RLS Policies (see full script in the original code or use these)
-- Organizations: employers can CRUD their own
CREATE POLICY "Employers can manage their own organizations"
  ON organizations FOR ALL
  USING (auth.uid() = employer_id);

-- Employees: employers can manage employees in their organizations
CREATE POLICY "Employers can manage employees in their organizations"
  ON employees FOR ALL
  USING (
    EXISTS (
      SELECT 1 FROM organizations
      WHERE organizations.id = employees.organization_id
        AND organizations.employer_id = auth.uid()
    )
  );

-- Roles: employers can manage roles in their organizations
CREATE POLICY "Employers can manage roles in their organizations"
  ON roles FOR ALL
  USING (
    EXISTS (
      SELECT 1 FROM organizations
      WHERE organizations.id = roles.organization_id
        AND organizations.employer_id = auth.uid()
    )
  );

-- Attendance: employees can only view their own records, employers can view all in their org
CREATE POLICY "Employees can view their own attendance"
  ON attendance FOR SELECT
  USING (auth.uid() = (SELECT auth.uid() FROM employees WHERE employees.id = attendance.employee_id));

-- (Optional) Employers can view all attendance in their org
CREATE POLICY "Employers can view all attendance in their org"
  ON attendance FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM organizations
      WHERE organizations.id = attendance.organization_id
        AND organizations.employer_id = auth.uid()
    )
  );

-- Allow employees to insert their own attendance (with RLS bypass via service key)
-- For simplicity, we use the service_role key for employee insert/update in the app.
```

### 3. Storage Bucket

1. In Supabase Dashboard → **Storage**, create a new bucket named `attendance-photos`.
2. Set bucket to **public** (or configure a policy to allow authenticated uploads).
3. Update bucket permissions to allow inserts/selects (the app uses the service role key for uploads).

### 4. Supabase Configuration in the App

- When you first open the app, click the **Settings** (⚙️) button on the login screen.
- Enter the admin password (default: `admin123` – can be changed later).
- Fill in:
  - **Supabase URL** – your project URL (e.g., `https://xxxxx.supabase.co`)
  - **Anon Key** – public anon key
  - **Service Role Key** – secret key (bypasses RLS, used for employee PIN verification and attendance inserts)
- Save. The app will now be connected to your Supabase project.

### 5. Running the App

- Simply open the `index.html` file in a browser, or serve it with any static server (Live Server, Netlify, etc.).
- All logic is client‑side; no build step required.

---

## 👥 Employee Setup (for Employers / Admins)

Before employees can use the app, you need to add them to the database.

### 1. Create an Organization

You can insert a record directly in Supabase (SQL Editor or Table Editor):

```sql
INSERT INTO organizations (employer_id, org_code, org_name)
VALUES ('<your-auth-user-id>', 'ACME123', 'Acme Corp');
```

- `employer_id` must be a valid `auth.users` ID (you can create a user via Supabase Auth).
- `org_code` is the unique code employees will use to log in.

### 2. Add Employees

For each employee, insert a record:

```sql
INSERT INTO employees (organization_id, employee_id, name, role, password_hash, pin_hash)
VALUES (
  '<organization-uuid>',
  '1234',                    -- employee_id (used as login PIN)
  'John Doe',
  'Developer',
  '',                        -- not used for PIN login, can be left empty
  '<sha256-hash-of-1234>'   -- generate with: SELECT encode(sha256('1234'::bytea), 'hex')
);
```

- `employee_id` is the **4‑digit PIN** the employee will enter.
- `pin_hash` must be the SHA‑256 digest of that PIN (the app computes this client‑side).

> 💡 **Tip:** You can generate the hash in SQL: `SELECT encode(sha256('1234'::bytea), 'hex')`

### 3. (Optional) Define Roles

If you want autocomplete suggestions for employee roles, add them to the `roles` table.

---

## 👤 Employee Usage Guide

### 1. Login

1. Open the app in a browser.
2. Enter your **Organization Code** (e.g., `ACME123`). The app will remember it for next time.
3. Click **Continue**.
4. The organization name will appear. Enter your **4‑digit PIN**.
5. Click **Unlock** (or press Enter). You’ll be taken to the dashboard.

### 2. Dashboard Overview

- **Employee name** and **organization name** are shown.
- **Status badge** indicates if you are currently “Punched In” or “Punched Out”.
- **Selfie preview area** – use the camera to capture a photo before punching.
- **Punch button** – big round button for punching in/out.
- **Last punch time** – shows the most recent punch action.
- **Monthly Report** button and **Logout** button.

### 3. Punch In

1. **Start Camera** – click the “📷 Start Camera” button and allow webcam access.
2. **Capture Selfie** – once the camera preview appears, click **Capture** to take a photo. The image will be stored in the app.
3. Tap the **Punch In** button (green). The selfie will be uploaded, and the attendance record will be saved.
4. The button will turn red and read **Punch Out** – you are now clocked in.

### 4. Punch Out

1. **Capture a selfie** again (you can start the camera and take a new photo).
2. Tap the **Punch Out** button (red).
3. A modal will appear asking for a **reason**:
   - **Office / Work** – select a specific sub‑reason (client meeting, field work, site visit, training, office errand, other).
   - **Personal** – no further details are required.
4. Click **Confirm Punch Out**. The selfie and reason are saved, and the punch‑out time is recorded.
5. The button resets to **Punch In** and the status updates.

### 5. Monthly Report

1. On the dashboard, click **📊 Monthly Report**.
2. You’ll see a calendar for the current month.
   - **Colors**: Green = Present, Red = Absent, Yellow = Half‑Day.
   - **Summary stats** show total Present, Absent, Half‑Day days.
3. Click any day in the calendar to view detailed punch times, reasons, and any additional info.
4. Use the arrow buttons (`‹` / `›`) to navigate to other months.

### 6. Logout

- Click the **🚪 Logout** button on the dashboard. Your session will be cleared, and you’ll return to the login screen.

---

## ⚙️ Settings Page (for Admins / Developers)

- Access via the **⚙️ Settings** button on the login screen.
- The page is password‑protected (default: `admin123`).
- You can update:
  - Supabase URL
  - Anon Key
  - Service Role Key
  - Admin password (change it to something secure)
- After saving, the app will reconnect to Supabase with the new credentials.

---

## 🔒 Security Considerations

- **PINs are hashed** with SHA‑256 on the client before transmission.
- **Service role key** is used for employee authentication and attendance writes – it bypasses RLS. Ensure this key is kept secret (stored only in the browser’s local storage, which is accessible only to the user).
- For production, consider implementing a proper authentication flow with Supabase Auth and JWT tokens, but this demo is designed for simplicity.

---

## 📦 File Structure

The entire application is contained in **a single HTML file**. All CSS and JavaScript are embedded, making deployment trivial.

```
index.html   # Full app
```

---

## 🐞 Troubleshooting

| Issue | Possible Solution |
|-------|-------------------|
| **“Supabase not configured”** | Go to Settings and enter valid Supabase URL and keys. |
| **Camera not starting** | Ensure browser has camera permissions. Try a different browser or allow camera access in site settings. |
| **“Organization not found”** | Double‑check the org code. It must match exactly (case‑insensitive). |
| **PIN rejected** | Make sure the employee record exists with the correct `employee_id` and `pin_hash`. You can re‑hash the PIN in SQL. |
| **Selfie upload fails** | Check that the `attendance-photos` bucket exists and is public. Also verify that the service role key has storage permissions. |
| **Report shows no data** | Ensure there are attendance records for the current month. If not, punch in/out to create records. |

---

## 🚀 Future Enhancements

- Real‑time location tracking during punches.
- Push notifications for shift reminders.
- Admin dashboard to manage all employees and attendance across the organization.
- OTP or biometric authentication for added security.

---

## 📄 License

MIT – free to use and modify.

---

## 🙌 Contributing

Feel free to fork and submit pull requests. For major changes, please open an issue first.
