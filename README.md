# intern_task2
from flask import Flask, render_template, request, redirect, url_for, session, flash
import sqlite3
from werkzeug.security import generate_password_hash, check_password_hash
import os

# Use 'templates' folder explicitly (optional)
app = Flask(_name_, template_folder='templates')
app.secret_key = 'supersecretkey'

# Initialize database
def init_db():
    conn = sqlite3.connect('database.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS admin (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    username TEXT UNIQUE NOT NULL,
                    password TEXT NOT NULL)''')
    c.execute('''CREATE TABLE IF NOT EXISTS employees (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    name TEXT NOT NULL,
                    email TEXT UNIQUE NOT NULL,
                    position TEXT NOT NULL,
                    salary REAL NOT NULL)''')
    # Create default admin
    c.execute("INSERT OR IGNORE INTO admin (id, username, password) VALUES (1, 'admin', ?)" ,
              (generate_password_hash('admin123'),))
    conn.commit()
    conn.close()

init_db()

# Get DB connection
def get_db():
    return sqlite3.connect('database.db')

@app.route('/')
def home():
    return redirect('/login')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        conn = get_db()
        c = conn.cursor()
        c.execute("SELECT * FROM admin WHERE username = ?", (username,))
        user = c.fetchone()
        conn.close()
        if user and check_password_hash(user[2], password):
            session['admin'] = username
            return redirect('/dashboard')
        else:
            flash('Invalid login credentials')
    return render_template('login.html')

@app.route('/logout')
def logout():
    session.pop('admin', None)
    return redirect('/login')

@app.route('/dashboard')
def dashboard():
    if 'admin' not in session:
        return redirect('/login')
    conn = get_db()
    c = conn.cursor()
    c.execute("SELECT * FROM employees")
    employees = c.fetchall()
    conn.close()
    return render_template('dashboard.html', employees=employees)

@app.route('/add', methods=['GET', 'POST'])
def add_employee():
    if 'admin' not in session:
        return redirect('/login')
    if request.method == 'POST':
        name = request.form['name']
        email = request.form['email']
        position = request.form['position']
        salary = request.form['salary']
        if not (name and email and position and salary):
            flash('All fields are required.')
            return redirect('/add')
        conn = get_db()
        c = conn.cursor()
        try:
            c.execute("INSERT INTO employees (name, email, position, salary) VALUES (?, ?, ?, ?)",
                      (name, email, position, float(salary)))
            conn.commit()
        except sqlite3.IntegrityError:
            flash('Email must be unique.')
        finally:
            conn.close()
        return redirect('/dashboard')
    return render_template('add_employee.html')

@app.route('/edit/<int:id>', methods=['GET', 'POST'])
def edit_employee(id):
    if 'admin' not in session:
        return redirect('/login')
    conn = get_db()
    c = conn.cursor()
    if request.method == 'POST':
        name = request.form['name']
        email = request.form['email']
        position = request.form['position']
        salary = request.form['salary']
        c.execute("UPDATE employees SET name=?, email=?, position=?, salary=? WHERE id=?",
                  (name, email, position, float(salary), id))
        conn.commit()
        conn.close()
        return redirect('/dashboard')
    c.execute("SELECT * FROM employees WHERE id=?", (id,))
    emp = c.fetchone()
    conn.close()
    return render_template('edit_employee.html', emp=emp)

@app.route('/delete/<int:id>')
def delete_employee(id):
    if 'admin' not in session:
        return redirect('/login')
    conn = get_db()
    c = conn.cursor()
    c.execute("DELETE FROM employees WHERE id=?", (id,))
    conn.commit()
    conn.close()
    return redirect('/dashboard')

if _name_ == '_main_':
    app.run(debug=True)

#HTML(TEMP)
<!DOCTYPE html>
<html>
<head><title>Add Employee</title></head>
<body>
  <h2>Add Employee</h2>
  <form method="post">
    Name: <input type="text" name="name" required><br><br>
    Email: <input type="email" name="email" required><br><br>
    Position: <input type="text" name="position" required><br><br>
    Salary: <input type="number" step="0.01" name="salary" required><br><br>
    <input type="submit" value="Add">
  </form>
  {% with messages = get_flashed_messages() %}
    {% if messages %}
      <p style="color:red;">{{ messages[0] }}</p>
    {% endif %}
  {% endwith %}
</body>
</html>

<!DOCTYPE html>
<html>
<head><title>Dashboard</title></head>
<body>
  <h2>Employee Dashboard</h2>
  <a href="/add">Add Employee</a> | <a href="/logout">Logout</a><br><br>

  <table border="1">
    <tr>
      <th>Name</th><th>Email</th><th>Position</th><th>Salary</th><th>Actions</th>
    </tr>
    {% for e in employees %}
    <tr>
      <td>{{ e[1] }}</td>
      <td>{{ e[2] }}</td>
      <td>{{ e[3] }}</td>
      <td>{{ e[4] }}</td>
      <td>
        <a href="/edit/{{ e[0] }}">Edit</a> |
        <a href="/delete/{{ e[0] }}">Delete</a>
      </td>
    </tr>
    {% endfor %}
  </table>
</body>
</html>

<!DOCTYPE html>
<html>
<head><title>Edit Employee</title></head>
<body>
  <h2>Edit Employee</h2>
  <form method="post">
    Name: <input type="text" name="name" value="{{ emp[1] }}" required><br><br>
    Email: <input type="email" name="email" value="{{ emp[2] }}" required><br><br>
    Position: <input type="text" name="position" value="{{ emp[3] }}" required><br><br>
    Salary: <input type="number" step="0.01" name="salary" value="{{ emp[4] }}" required><br><br>
    <input type="submit" value="Update">
  </form>
</body>
</html>

<!DOCTYPE html>
<html>
<head><title>Login</title></head>
<body>
  <h2>Admin Login</h2>
  <form method="post">
    <input type="text" name="username" placeholder="Username" required><br><br>
    <input type="password" name="password" placeholder="Password" required><br><br>
    <input type="submit" value="Login">
  </form>
  {% with messages = get_flashed_messages() %}
    {% if messages %}
      <p style="color:red;">{{ messages[0] }}</p>
    {% endif %}
  {% endwith %}
</body>
</html>
