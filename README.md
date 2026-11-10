import os
import sqlite3
import random
import string
import time
from flask import Flask, render_template_string, request, redirect, session, flash, send_from_directory
from werkzeug.security import generate_password_hash, check_password_hash
from werkzeug.utils import secure_filename
from flask_mail import Mail, Message

app = Flask(__name__)
app.secret_key = os.environ.get("FLASK_SECRET_KEY", "edutrack-super-secret-key-999")

UPLOAD_FOLDER = "uploads"
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER

# --- GMAIL SMTP CONFIGURATION (Real OTP Emails) ---
# अपनी असली ईमेल और 16-अक्षरों का Google App Password यहाँ सेट करें
app.config['MAIL_SERVER'] = 'smtp.gmail.com'
app.config['MAIL_PORT'] = 587
app.config['MAIL_USE_TLS'] = True
app.config['MAIL_USERNAME'] = os.environ.get('MAIL_USERNAME', 'your_email@gmail.com')
app.config['MAIL_PASSWORD'] = os.environ.get('MAIL_PASSWORD', 'your_google_app_password')
app.config['MAIL_DEFAULT_SENDER'] = ('EduTrack Pro', app.config['MAIL_USERNAME'])

mail = Mail(app)

# अस्थाई रजिस्ट्रेशन डेटा (OTP Verify होने तक)
PENDING_REGISTRATIONS = {}

# --- DATABASE SETUP ---
def get_db():
    conn = sqlite3.connect("classroom.db")
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = get_db()
    conn.execute('''CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT, 
        name TEXT, 
        email TEXT UNIQUE, 
        password TEXT, 
        role TEXT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )''')
    conn.execute('''CREATE TABLE IF NOT EXISTS classrooms (
        id INTEGER PRIMARY KEY AUTOINCREMENT, 
        name TEXT, 
        subject TEXT, 
        code TEXT UNIQUE, 
        teacher_id INTEGER
    )''')
    conn.execute('''CREATE TABLE IF NOT EXISTS enrollments (
        id INTEGER PRIMARY KEY AUTOINCREMENT, 
        classroom_id INTEGER, 
        student_id INTEGER
    )''')
    conn.execute('''CREATE TABLE IF NOT EXISTS attendance (
        id INTEGER PRIMARY KEY AUTOINCREMENT, 
        classroom_id INTEGER, 
        student_id INTEGER, 
        date TEXT, 
        status TEXT
    )''')
    conn.execute('''CREATE TABLE IF NOT EXISTS resources (
        id INTEGER PRIMARY KEY AUTOINCREMENT, 
        classroom_id INTEGER, 
        title TEXT, 
        filename TEXT, 
        date TEXT
    )''')
    conn.execute('''CREATE TABLE IF NOT EXISTS absence_requests (
        id INTEGER PRIMARY KEY AUTOINCREMENT, 
        classroom_id INTEGER, 
        student_id INTEGER, 
        date TEXT, 
        reason TEXT, 
        status TEXT DEFAULT 'Pending'
    )''')
    conn.commit()
    conn.close()

init_db()

# --- HIGH-TECH UI SHELL ---
BASE_TEMPLATE = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>EduTrack Pro | NextGen Classroom</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800&display=swap" rel="stylesheet">
    <style>
        :root {
            --primary: #4f46e5;
            --primary-hover: #4338ca;
            --bg: #0f172a;
            --card-bg: rgba(30, 41, 59, 0.7);
            --border: rgba(255, 255, 255, 0.08);
            --text-main: #f8fafc;
            --text-muted: #94a3b8;
        }
        body {
            font-family: 'Plus Jakarta Sans', sans-serif;
            background: radial-gradient(circle at 15% 50%, rgba(79, 70, 229, 0.15) 0%, transparent 40%),
                        radial-gradient(circle at 85% 30%, rgba(14, 165, 233, 0.15) 0%, transparent 40%),
                        var(--bg);
            color: var(--text-main);
            min-height: 100vh;
        }
        .navbar-custom {
            background: rgba(15, 23, 42, 0.85);
            backdrop-filter: blur(16px);
            border-bottom: 1px solid var(--border);
        }
        .glass-card {
            background: var(--card-bg);
            backdrop-filter: blur(12px);
            border: 1px solid var(--border);
            border-radius: 18px;
            box-shadow: 0 10px 30px -10px rgba(0, 0, 0, 0.5);
        }
        .btn-tech {
            background: linear-gradient(135deg, #6366f1, #4f46e5);
            color: white;
            border: none;
            border-radius: 12px;
            font-weight: 600;
            padding: 10px 20px;
            transition: all 0.3s;
        }
        .btn-tech:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 20px rgba(99, 102, 241, 0.4);
            color: white;
        }
        .form-control, .form-select {
            background: rgba(15, 23, 42, 0.6);
            border: 1px solid rgba(255, 255, 255, 0.12);
            color: #fff;
            border-radius: 12px;
            padding: 12px;
        }
        .form-control:focus, .form-select:focus {
            background: rgba(15, 23, 42, 0.85);
            border-color: #6366f1;
            box-shadow: 0 0 0 4px rgba(99, 102, 241, 0.15);
            color: #fff;
        }
        .badge-code {
            background: rgba(99, 102, 241, 0.15);
            border: 1px solid rgba(99, 102, 241, 0.4);
            color: #a5b4fc;
            padding: 4px 10px;
            border-radius: 8px;
            font-family: monospace;
            font-size: 0.95rem;
        }
    </style>
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark navbar-custom px-4 py-3 sticky-top">
        <div class="container-fluid">
            <a class="navbar-brand fw-bold fs-4 d-flex align-items-center gap-2" href="/dashboard">
                <i class="fa-solid fa-graduation-cap text-primary"></i> EduTrack <span class="badge bg-primary fs-6">Pro</span>
            </a>
            <div>
                {% if session.get('user') %}
                    <span class="me-3 text-secondary">
                        <i class="fa-solid fa-user-circle me-1"></i> {{ session['user']['name'] }} 
                        <span class="badge bg-secondary ms-1">{{ session['user']['role'] }}</span>
                    </span>
                    <a href="/logout" class="btn btn-outline-danger btn-sm rounded-pill px-3">
                        <i class="fa-solid fa-arrow-right-from-bracket me-1"></i> Logout
                    </a>
                {% endif %}
            </div>
        </div>
    </nav>
    <div class="container py-4">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert alert-{{ category if category != 'message' else 'info' }} alert-dismissible fade show border-0 rounded-4 shadow-sm" role="alert">
                        {{ message }}
                        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                    </div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        {{ content|safe }}
    </div>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
"""

def render_page(inner_content, **context):
    body = render_template_string(inner_content, **context)
    return render_template_string(BASE_TEMPLATE, content=body)

# --- AUTH ROUTES ---
@app.route("/", methods=["GET", "POST"])
def login():
    if 'user' in session:
        return redirect("/dashboard")
        
    if request.method == "POST":
        email = request.form['email'].strip().lower()
        password = request.form['password']
        
        conn = get_db()
        user = conn.execute("SELECT * FROM users WHERE email = ?", (email,)).fetchone()
        conn.close()
        
        if user and check_password_hash(user['password'], password):
            session['user'] = {
                'id': user['id'],
                'name': user['name'],
                'email': user['email'],
                'role': user['role']
            }
            flash(f"Welcome back, {user['name']}!", "success")
            return redirect("/dashboard")
        flash("Invalid email or password. Please verify your credentials.", "danger")
        
    return render_page("""
    <div class="row justify-content-center pt-4">
        <div class="col-md-5">
            <div class="glass-card p-4 p-md-5">
                <div class="text-center mb-4">
                    <h3 class="fw-bold">Welcome Back</h3>
                    <p class="text-secondary small">Access your smart classroom portal</p>
                </div>
                <form method="post">
                    <div class="mb-3">
                        <label class="form-label text-secondary small">Verified Email</label>
                        <input name="email" type="email" class="form-control" placeholder="name@example.com" required>
                    </div>
                    <div class="mb-4">
                        <label class="form-label text-secondary small">Password</label>
                        <input name="password" type="password" class="form-control" placeholder="Enter password" required>
                    </div>
                    <button class="btn btn-tech w-100 py-3">Sign In</button>
                </form>
                <div class="text-center mt-4">
                    <span class="text-secondary small">Don't have an account?</span> 
                    <a href="/register" class="text-decoration-none fw-semibold text-primary">Create Account</a>
                </div>
            </div>
        </div>
    </div>
    """)

@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        name = request.form['name'].strip()
        email = request.form['email'].strip().lower()
        password = request.form['password']
        role = request.form['role']
        
        # पासवर्ड स्ट्रेंथ चेक
        if len(password) < 6:
            flash("Password must be at least 6 characters long.", "warning")
            return redirect("/register")
            
        conn = get_db()
        exists = conn.execute("SELECT id FROM users WHERE email = ?", (email,)).fetchone()
        conn.close()
        
        if exists:
            flash("An account with this email already exists.", "warning")
            return redirect("/register")
            
        # 6-अक्षरों का OTP जनरेट करें
        otp = str(random.randint(100000, 999999))
        PENDING_REGISTRATIONS[email] = {
            'name': name,
            'email': email,
            'password': generate_password_hash(password),
            'role': role,
            'otp': otp,
            'timestamp': time.time()
        }
        
        # ईमेल द्वारा OTP भेजें
        try:
            msg = Message("EduTrack Pro - Email Verification Code", recipients=[email])
            msg.body = f"Hello {name},\n\nYour 6-digit verification code is: {otp}\n\nThis OTP is valid for 10 minutes.\n\nTeam EduTrack"
            mail.send(msg)
            session['verify_email'] = email
            flash("Verification OTP sent! Check your email inbox or spam folder.", "info")
            return redirect("/verify-otp")
        except Exception as e:
            # लोकल टेस्टिंग सुविधा: अगर SMTP क्रेडेंशियल न जुड़े हों
            session['verify_email'] = email
            flash(f"[DEV MODE] SMTP not fully configured. Your test OTP is: {otp}", "warning")
            return redirect("/verify-otp")
            
    return render_page("""
    <div class="row justify-content-center pt-3">
        <div class="col-md-5">
            <div class="glass-card p-4 p-md-5">
                <div class="text-center mb-4">
                    <h3 class="fw-bold">Create Account</h3>
                    <p class="text-secondary small">Real-time OTP verification required</p>
                </div>
                <form method="post">
                    <div class="mb-3">
                        <label class="form-label text-secondary small">Full Name</label>
                        <input name="name" class="form-control" placeholder="e.g. John Doe" required>
                    </div>
                    <div class="mb-3">
                        <label class="form-label text-secondary small">Real Email Address</label>
                        <input name="email" type="email" class="form-control" placeholder="name@domain.com" required>
                    </div>
                    <div class="mb-3">
                        <label class="form-label text-secondary small">Password</label>
                        <input name="password" type="password" class="form-control" placeholder="Min. 6 characters" required>
                    </div>
                    <div class="mb-4">
                        <label class="form-label text-secondary small">Role</label>
                        <select name="role" class="form-select">
                            <option value="Teacher">Teacher / Educator</option>
                            <option value="Student">Student</option>
                        </select>
                    </div>
                    <button class="btn btn-tech w-100 py-3">Send Verification Code</button>
                </form>
                <div class="text-center mt-4">
                    <span class="text-secondary small">Already registered?</span> 
                    <a href="/" class="text-decoration-none fw-semibold text-primary">Login</a>
                </div>
            </div>
        </div>
    </div>
    """)

@app.route("/verify-otp", methods=["GET", "POST"])
def verify_otp():
    email = session.get('verify_email')
    if not email or email not in PENDING_REGISTRATIONS:
        flash("Registration session expired. Please register again.", "warning")
        return redirect("/register")
        
    if request.method == "POST":
        user_otp = request.form['otp'].strip()
        data = PENDING_REGISTRATIONS[email]
        
        # OTP एक्सपायरी: 10 मिनट
        if time.time() - data['timestamp'] > 600:
            del PENDING_REGISTRATIONS[email]
            flash("OTP expired. Please register again.", "danger")
            return redirect("/register")
            
        if user_otp == data['otp']:
            conn = get_db()
            conn.execute("INSERT INTO users (name, email, password, role) VALUES (?, ?, ?, ?)",
                         (data['name'], data['email'], data['password'], data['role']))
            conn.commit()
            conn.close()
            
            del PENDING_REGISTRATIONS[email]
            session.pop('verify_email', None)
            flash("Email verified successfully! You can now login.", "success")
            return redirect("/")
        else:
            flash("Incorrect OTP. Please enter the valid 6-digit code.", "danger")
            
    return render_page("""
    <div class="row justify-content-center pt-4">
        <div class="col-md-5">
            <div class="glass-card p-4 p-md-5 text-center">
                <div class="mb-3 text-primary fs-1">
                    <i class="fa-solid fa-envelope-circle-check"></i>
                </div>
                <h3 class="fw-bold">Enter Verification Code</h3>
                <p class="text-secondary small">Enter the 6-digit code sent to <b>{{ email }}</b></p>
                <form method="post" class="mt-4">
                    <div class="mb-4">
                        <input name="otp" class="form-control text-center fs-3 tracking-widest" maxlength="6" placeholder="000000" autofocus required>
                    </div>
                    <button class="btn btn-tech w-100 py-3">Verify & Activate Account</button>
                </form>
            </div>
        </div>
    </div>
    """, email=email)

@app.route("/logout")
def logout():
    session.clear()
    flash("You have been signed out.", "info")
    return redirect("/")

# --- DASHBOARD ---
@app.route("/dashboard")
def dashboard():
    if 'user' not in session:
        return redirect("/")
    u = session['user']
    conn = get_db()
    if u['role'] == "Teacher":
        rooms = conn.execute("SELECT * FROM classrooms WHERE teacher_id=?", (u['id'],)).fetchall()
    else:
        rooms = conn.execute("SELECT c.* FROM classrooms c JOIN enrollments e ON c.id=e.classroom_id WHERE e.student_id=?", (u['id'],)).fetchall()
    conn.close()
    
    return render_page("""
    <div class="d-flex justify-content-between align-items-center mb-4">
        <div>
            <h2 class="fw-bold mb-0">Control Center</h2>
            <p class="text-secondary small">Manage enrolled courses and learning materials</p>
        </div>
        <button class="btn btn-tech" data-bs-toggle="modal" data-bs-target="#actionModal">
            {% if u['role'] == 'Teacher' %}
                <i class="fa-solid fa-plus me-1"></i> New Classroom
            {% else %}
                <i class="fa-solid fa-key me-1"></i> Join with Code
            {% endif %}
        </button>
    </div>

    <!-- Modal for Action -->
    <div class="modal fade" id="actionModal" tabindex="-1">
        <div class="modal-dialog modal-dialog-centered">
            <div class="modal-content glass-card" style="background: #1e293b;">
                <div class="modal-header border-0">
                    <h5 class="modal-title fw-bold">{% if u['role'] == 'Teacher' %}Create Classroom{% else %}Join Classroom{% endif %}</h5>
                    <button type="button" class="btn-close btn-close-white" data-bs-dismiss="modal"></button>
                </div>
                <div class="modal-body">
                    {% if u['role'] == 'Teacher' %}
                        <form method="post" action="/create_class">
                            <input name="name" class="form-control mb-3" placeholder="Class Name (e.g. CS50 AI)" required>
                            <input name="subject" class="form-control mb-4" placeholder="Subject / Discipline" required>
                            <button class="btn btn-tech w-100">Initialize Classroom</button>
                        </form>
                    {% else %}
                        <form method="post" action="/join_class">
                            <input name="code" class="form-control text-uppercase text-center fs-4 mb-4" placeholder="6-DIGIT CODE" required>
                            <button class="btn btn-tech w-100">Enroll Course</button>
                        </form>
                    {% endif %}
                </div>
            </div>
        </div>
    </div>

    <div class="row g-4">
        {% for r in rooms %}
            <div class="col-md-4">
                <div class="glass-card p-4 h-100 d-flex flex-column justify-content-between">
                    <div>
                        <div class="d-flex justify-content-between align-items-start mb-2">
                            <span class="badge bg-secondary">{{ r['subject'] }}</span>
                            <span class="badge-code">{{ r['code'] }}</span>
                        </div>
                        <h4 class="fw-bold mb-3">{{ r['name'] }}</h4>
                    </div>
                    <a href="/classroom/{{ r['id'] }}" class="btn btn-outline-light rounded-pill w-100 mt-3">
                        Launch Classroom <i class="fa-solid fa-arrow-right ms-1"></i>
                    </a>
                </div>
            </div>
        {% else %}
            <div class="col-12 text-center py-5">
                <i class="fa-solid fa-folder-open fs-1 text-secondary mb-3"></i>
                <p class="text-secondary">No classrooms found. Get started by clicking the button above.</p>
            </div>
        {% endfor %}
    </div>
    """, u=u, rooms=rooms)

@app.route("/cre