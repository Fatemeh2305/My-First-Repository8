my_flask_app/
│
├── app.py
│   └──
        import sqlite3
        from flask import Flask, render_template, request, g, redirect, url_for, session, flashs
        from werkzeug.security import generate_password_hash, check_password_hashs

        app = Flask(__name__)
        app.secret_key = "supersecretkey"  # change in production
        DATABASE = "app.db"

        def get_db():
            db = getattr(g, "_database", None)
            if db is None:
                db = g._database = sqlite3.connect(DATABASE)
                db.row_factory = sqlite3.Row
            return db

        @app.teardown_appcontextS
        def close_connection(exception):
            db = getattr(g, "_database", None)
            if db is not None:
                db.close()

        def init_db():
            with app.app_context():
                db = get_db()
                cursor = db.cursor()
                cursor.execute("""
                    CREATE TABLE IF NOT EXISTS users (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        username TEXT UNIQUE NOT NULL,
                        password TEXT NOT NULL
                    )
                """)
                cursor.execute("""
                    CREATE TABLE IF NOT EXISTS messages (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        name TEXT NOT NULL,
                        email TEXT NOT NULL,
                        message TEXT NOT NULL
                    )
                """)
                db.commit()

        @app.route("/")
        def home():
            return render_template("index.html", title="Home")

        @app.route("/about")
        def about():
            return render_template("about.html", title="About")

        @app.route("/contact", methods=["GET", "POST"])
        def contact():
            if request.method == "POST":
                name = request.form.get("name")
                email = request.form.get("email")
                message = request.form.get("message")

                db = get_db()
                cursor = db.cursor()
                cursor.execute("INSERT INTO messages (name, email, message) VALUES (?, ?, ?)",
                               (name, email, message))
                db.commit()

                return render_template("contact.html", title="Contact", success=True, name=name)
            return render_template("contact.html", title="Contact", success=False)

        @app.route("/register", methods=["GET", "POST"])
        def register():
            if request.method == "POST":
                username = request.form.get("username")
                password = request.form.get("password")
                hashed_pw = generate_password_hash(password)

                db = get_db()
                cursor = db.cursor()
                try:
                    cursor.execute("INSERT INTO users (username, password) VALUES (?, ?)", (username, hashed_pw))
                    db.commit()
                    flash("Registration successful! Please log in.", "success")
                    return redirect(url_for("login"))
                except sqlite3.IntegrityError:
                    flash("Username already exists!", "danger")
            return render_template("register.html", title="Register")

        @app.route("/login", methods=["GET", "POST"])
        def login():
            if request.method == "POST":
                username = request.form.get("username")
                password = request.form.get("password")

                db = get_db()
                cursor = db.cursor()
                cursor.execute("SELECT * FROM users WHERE username=?", (username,))
                user = cursor.fetchone()

                if user and check_password_hash(user["password"], password):
                    session["logged_in"] = True
                    session["username"] = username
                    flash("Login successful!", "success")
                    return redirect(url_for("admin"))
                else:
                    flash("Invalid credentials", "danger")
            return render_template("login.html", title="Login")

        @app.route("/logout")
        def logout():
            session.clear()
            flash("You have been logged out.", "info")
            return redirect(url_for("home"))

        @app.route("/admin")
        def admin():
            if not session.get("logged_in"):
                flash("Please log in to access admin dashboard.", "warning")
                return redirect(url_for("login"))

            db = get_db()
            cursor = db.cursor()
            cursor.execute("SELECT * FROM messages ORDER BY id DESC")
            messages = cursor.fetchall()
            return render_template("admin.html", title="Admin Dashboard", messages=messages)

        if __name__ == "__main__":
            init_db()
            app.run(debug=True)

├── requirements.txt
│   └──
        Flask
        Werkzeug

├── templates/
│   ├── base.html
│   │   └──
                <!DOCTYPE html>
                <html lang="en">
                <head>
                    <meta charset="UTF-8">
                    <meta name="viewport" content="width=device-width, initial-scale=1.0">
                    <title>{{ title if title else "Flask App" }}</title>
                    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
                </head>
                <body>
                    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
                        <div class="container">
                            <a class="navbar-brand" href="/">Flask App</a>
                            <div>
                                <a class="nav-link d-inline text-white" href="/">Home</a>
                                <a class="nav-link d-inline text-white" href="/about">About</a>
                                <a class="nav-link d-inline text-white" href="/contact">Contact</a>
                                {% if session.get("logged_in") %}
                                    <a class="nav-link d-inline text-warning" href="/admin">Admin</a>
                                    <a class="nav-link d-inline text-danger" href="/logout">Logout</a>
                                {% else %}
                                    <a class="nav-link d-inline text-info" href="/login">Login</a>
                                    <a class="nav-link d-inline text-success" href="/register">Register</a>
                                {% endif %}
                            </div>
                        </div>
                    </nav>
                    <div class="container mt-4">
                        {% with messages = get_flashed_messages(with_categories=true) %}
                          {% if messages %}
                            {% for category, message in messages %}
                              <div class="alert alert-{{ category }}">{{ message }}</div>
                            {% endfor %}
                          {% endif %}
                        {% endwith %}
                        {% block content %}{% endblock %}
                    </div>
                </body>
                </html>
│
│   ├── index.html
│   │   └──
                {% extends "base.html" %}
                {% block content %}
                <h1>Welcome to the Flask App</h1>
                <p class="lead">Now with secure user authentication and an admin dashboard.</p>
                {% endblock %}
│
│   ├── about.html
│   │   └──
                {% extends "base.html" %}
                {% block content %}
                <h1>About</h1>
                <p>This app demonstrates Flask with authentication, database integration, and Bootstrap styling.</p>
                {% endblock %}
│
│   ├── contact.html
│   │   └──
                {% extends "base.html" %}
                {% block content %}
                <h1>Contact Us</h1>
                {% if success %}
                    <div class="alert alert-success">Thank you, {{ name }}! Your message has been saved.</div>
                {% else %}
                    <form method="POST">
                        <div class="mb-3">
                            <label class="form-label">Name</label>
                            <input type="text" class="form-control" name="name" required>
                        </div>
                        <div class="mb-3">
                            <label class="form-label">Email</label>
                            <input type="email" class="form-control" name="email" required>
                        </div>
                        <div class="mb-3">
                            <label class="form-label">Message</label>
                            <textarea class="form-control" name="message" rows="4" required></textarea>
                        </div>
                        <button type="submit" class="btn btn-primary">Send</button>
                    </form>
                {% endif %}
                {% endblock %}
│
│   ├── login.html
│   │   └──
                {% extends "base.html" %}
                {% block content %}
                <h1>Login</h1>
                <form method="POST">
                    <div class="mb-3">
                        <label class="form-label">Username</label>
                        <input type="text" class="form-control" name="username" required>
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Password</label>
                        <input type="password" class="form-control" name="password" required>
                    </div>
                    <button type="submit" class="btn btn-success">Login</button>
                </form>
                {% endblock %}
│
│   ├── register.html
│   │   └──
                {% extends "base.html" %}
                {% block content %}
                <h1>Register</h1>
                <form method="POST">
                    <div class="mb-3">
                        <label class="form-label">Username</label>
                        <input type="text" class="form-control" name="username" required>
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Password</label>
                        <input type="password" class="form-control" name="password" required>
                    </div>
                    <button type="submit" class="btn btn-primary">Register</button>
                </form>
                {% endblock %}
│
│   └── admin.html
│       └──
                {% extends "base.html" %}
                {% block content %}
                <h1>Admin Dashboard</h1>
                {% if messages %}
                    <table class="table table-bordered">
                        <thead>
                            <tr>
                                <th>ID</th>
                                <th>Name</th>
                                <th>Email</th>
                                <th>Message</th>
                            </tr>
                        </thead>
                        <tbody>
                            {% for msg in messages %}
                                <tr>
                                    <td>{{ msg.id }}</td>
                                    <td>{{ msg.name }}</td>
                                    <td>{{ msg.email }}</td>
                                    <td>{{ msg.message }}</td>
                                </tr>
                            {% endfor %}
                        </tbody>
                    </table>
                {% else %}
                    <p>No messages found.</p>
                {% endif %}
                {% endblock %}

└── README.md
    └──
        # Flask Web App with User Authentication

        Features:
        - User registration and login with hashed passwords
        - Contact form saves messages to SQLite database
        - Admin dashboard (protected by login)
        - Bootstrap 5 styling
        - Secure session handling

        ## Installation
        ```bash
        pip install -r requirements.txt
        python app.py
        ```

        Then visit:
        ```
        http://127.0.0.1:5000
        ```

