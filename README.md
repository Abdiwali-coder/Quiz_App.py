# Quiz_App.py
import sqlite3
import tkinter as tk
from tkinter import messagebox
from flask import Flask, jsonify, request
import threading
import requests

# Initialize the Flask app
app = Flask(__name__)

# Database setup
DATABASE = "quiz_app.db"

def init_db():
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()

    # Create tables
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS questions (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        question TEXT NOT NULL,
        correct_answer TEXT NOT NULL,
        incorrect_answers TEXT NOT NULL
    )
    """)

    cursor.execute("""
    CREATE TABLE IF NOT EXISTS user_scores (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT NOT NULL UNIQUE,
        score INTEGER DEFAULT 0
    )
    """)

    # Insert sample questions
    cursor.executemany("""
    INSERT INTO questions (question, correct_answer, incorrect_answers)
    VALUES (?, ?, ?)
    """, [
        ("What is the output of print('Hello' + 'World')?", "HelloWorld", "Hello World,Hello + World,Error"),
        ("Which keyword is used to define a function in Python?", "def", "function,fn,func"),
        ("What is the data type of 3.14?", "float", "int,str,bool"),
        ("What is the purpose of the 'if' statement?", "To make decisions", "To define a function,To repeat a block of code,To create a loop"),
        ("Which operator is used to concatenate two strings?", "+", "-,*,/")
    ])
    conn.commit()
    conn.close()

# Flask API endpoints
@app.route('/api/questions', methods=['GET'])
def get_questions():
    conn = sqlite3.connect(DATABASE)
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM questions")
    questions = cursor.fetchall()
    conn.close()
    return jsonify([dict(q) for q in questions])

@app.route('/api/submit', methods=['POST'])
def submit_answer():
    data = request.json
    username = data['username']
    question_id = data['question_id']
    user_answer = data['answer']

    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()

    # Check the answer
    cursor.execute("SELECT correct_answer FROM questions WHERE id = ?", (question_id,))
    correct_answer = cursor.fetchone()["correct_answer"]

    correct = user_answer == correct_answer
    score_increment = 1 if correct else 0

    # Update or insert user score
    cursor.execute("""
    INSERT INTO user_scores (username, score)
    VALUES (?, ?)
    ON CONFLICT(username) DO UPDATE SET score = score + ?
    """, (username, score_increment, score_increment))
    conn.commit()
    conn.close()

    return jsonify({"correct": correct, "score_increment": score_increment})

# Start Flask server in a separate thread
def start_flask():
    app.run(debug=False, use_reloader=False)

# Tkinter GUI Application
class QuizGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("Quiz Application")
        self.questions = []
        self.current_question_index = 0
        self.score = 0
        self.username = None

        # Welcome Screen
        self.intro_frame = tk.Frame(root)
        self.intro_label = tk.Label(self.intro_frame, text="Enter your username to start:", font=("Arial", 14))
        self.intro_label.pack(pady=10)
        self.username_entry = tk.Entry(self.intro_frame, font=("Arial", 14))
        self.username_entry.pack(pady=10)
        self.start_button = tk.Button(self.intro_frame, text="Start Quiz", command=self.start_quiz, font=("Arial", 14))
        self.start_button.pack(pady=10)
        self.intro_frame.pack(pady=20)

    def start_quiz(self):
        self.username = self.username_entry.get()
        if not self.username:
            messagebox.showwarning("Input Error", "Please enter your username!")
            return

        self.intro_frame.pack_forget()
        self.quiz_frame = tk.Frame(self.root)
        self.quiz_frame.pack(pady=20)
        self.load_questions()

    def load_questions(self):
        try:
            response = requests.get("http://127.0.0.1:5000/api/questions")
            if response.status_code == 200:
                self.questions = response.json()
                self.display_question()
            else:
                messagebox.showerror("Error", "Failed to load questions!")
        except requests.ConnectionError:
            messagebox.showerror("Error", "Unable to connect to the server.")

    def display_question(self):
        question_data = self.questions[self.current_question_index]
        question_text = question_data['question']

        self.question_label = tk.Label(self.quiz_frame, text=question_text, font=("Arial", 14), wraplength=400)
        self.question_label.pack(pady=10)

        self.answer_entry = tk.Entry(self.quiz_frame, font=("Arial", 14))
        self.answer_entry.pack(pady=10)

        self.submit_button = tk.Button(self.quiz_frame, text="Submit", command=self.submit_answer, font=("Arial", 14))
        self.submit_button.pack(pady=10)

    def submit_answer(self):
        user_answer = self.answer_entry.get()
        if not user_answer:
            messagebox.showwarning("Input Error", "Please enter your answer!")
            return

        question_data = self.questions[self.current_question_index]
        try:
            response = requests.post(
                "http://127.0.0.1:5000/api/submit",
                json={"username": self.username, "question_id": question_data['id'], "answer": user_answer}
            )
            if response.status_code == 200:
                result = response.json()
                if result['correct']:
                    messagebox.showinfo("Correct!", "Your answer is correct!")
                    self.score += result['score_increment']
                else:
                    messagebox.showinfo("Incorrect", f"The correct answer was: {question_data['correct_answer']}")

                self.next_question()
            else:
                messagebox.showerror("Error", "Failed to submit answer!")
        except requests.ConnectionError:
            messagebox.showerror("Error", "Unable to connect to the server.")

    def next_question(self):
        self.current_question_index += 1
        if self.current_question_index < len(self.questions):
            self.quiz_frame.destroy()
            self.quiz_frame = tk.Frame(self.root)
            self.quiz_frame.pack(pady=20)
            self.display_question()
        else:
            messagebox.showinfo("Quiz Complete", f"Your final score is: {self.score}")
            self.root.destroy()

# Initialize the app
if __name__ == "__main__":
    init_db()

    # Start Flask server in a separate thread
    flask_thread = threading.Thread(target=start_flask, daemon=True)
    flask_thread.start()

    # Start Tkinter GUI
    root = tk.Tk()
    app = QuizGUI(root)
    root.mainloop()
