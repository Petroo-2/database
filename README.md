# database
-- Database: library_management

-- Drop tables if they exist to start fresh
DROP TABLE IF EXISTS book_authors;
DROP TABLE IF EXISTS book_genres;
DROP TABLE IF EXISTS loan_records;
DROP TABLE IF EXISTS books;
DROP TABLE IF EXISTS authors;
DROP TABLE IF EXISTS genres;
DROP TABLE IF EXISTS members;

-- Table: members
CREATE TABLE members (
    member_id INT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone_number VARCHAR(20),
    registration_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Table: authors
CREATE TABLE authors (
    author_id INT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL
);

-- Table: genres
CREATE TABLE genres (
    genre_id INT PRIMARY KEY AUTO_INCREMENT,
    genre_name VARCHAR(50) UNIQUE NOT NULL
);

-- Table: books
CREATE TABLE books (
    book_id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(255) NOT NULL,
    isbn VARCHAR(20) UNIQUE NOT NULL,
    publication_year INT,
    publisher VARCHAR(100),
    genre_id INT,
    FOREIGN KEY (genre_id) REFERENCES genres(genre_id)
);

-- Table: book_authors (Many-to-Many relationship between books and authors)
CREATE TABLE book_authors (
    book_id INT,
    author_id INT,
    PRIMARY KEY (book_id, author_id),
    FOREIGN KEY (book_id) REFERENCES books(book_id),
    FOREIGN KEY (author_id) REFERENCES authors(author_id)
);

-- Table: loan_records
CREATE TABLE loan_records (
    loan_id INT PRIMARY KEY AUTO_INCREMENT,
    book_id INT NOT NULL,
    member_id INT NOT NULL,
    loan_date DATE NOT NULL,
    return_date DATE,
    due_date DATE NOT NULL,
    FOREIGN KEY (book_id) REFERENCES books(book_id),
    FOREIGN KEY (member_id) REFERENCES members(member_id)
);

-- Sample Data

-- Members
INSERT INTO members (first_name, last_name, email, phone_number) VALUES
('Alice', 'Smith', 'alice.smith@example.com', '123-456-7890'),
('Bob', 'Johnson', 'bob.johnson@example.com', '987-654-3210'),
('Charlie', 'Brown', 'charlie.brown@example.com', '555-123-4567');

-- Authors
INSERT INTO authors (first_name, last_name) VALUES
('Jane', 'Austen'),
('George', 'Orwell'),
('J.R.R.', 'Tolkien');

-- Genres
INSERT INTO genres (genre_name) VALUES
('Fiction'),
('Science Fiction'),
('Fantasy');

-- Books
INSERT INTO books (title, isbn, publication_year, publisher, genre_id) VALUES
('Pride and Prejudice', '978-0141439518', 1813, 'Penguin Classics', 1),
('Nineteen Eighty-Four', '978-0451524935', 1949, 'Signet Classics', 2),
('The Hobbit', '978-0547928227', 1937, 'Houghton Mifflin', 3);

-- Book Authors
INSERT INTO book_authors (book_id, author_id) VALUES
(1, 1),
(2, 2),
(3, 3);

-- Loan Records
INSERT INTO loan_records (book_id, member_id, loan_date, due_date) VALUES
(1, 2, '2025-05-01', '2025-05-15'),
(3, 1, '2025-05-03', '2025-05-17');

-- Database: task_manager

DROP TABLE IF EXISTS tasks;

CREATE TABLE tasks (
    task_id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    due_date DATE,
    status ENUM('To Do', 'In Progress', 'Done') DEFAULT 'To Do',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);





from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import mysql.connector
from typing import Optional

app = FastAPI()

# Database Configuration
DB_CONFIG = {
    'host': 'localhost',  # Replace with your MySQL host
    'user': 'your_user',  # Replace with your MySQL username
    'password': 'your_password',  # Replace with your MySQL password
    'database': 'task_manager'  # Replace with your database name
}

def get_db():
    try:
        db = mysql.connector.connect(**DB_CONFIG)
        return db
    except mysql.connector.Error as err:
        raise HTTPException(status_code=500, detail=f"Database connection error: {err}")

class TaskBase(BaseModel):
    title: str
    description: Optional[str] = None
    due_date: Optional[str] = None
    status: Optional[str] = "To Do"

class TaskCreate(TaskBase):
    pass

class TaskUpdate(TaskBase):
    pass

class Task(TaskBase):
    task_id: int
    created_at: datetime
    updated_at: datetime






    class Config:
        orm_mode = True

from datetime import datetime

# CRUD Operations

@app.post("/tasks/", response_model=Task, status_code=201)
def create_task(task: TaskCreate):
    db = get_db()
    cursor = db.cursor()
    query = "INSERT INTO tasks (title, description, due_date, status) VALUES (%s, %s, %s, %s)"
    values = (task.title, task.description, task.due_date, task.status)
    cursor.execute(query, values)
    db.commit()
    task_id = cursor.lastrowid
    cursor.close()
    db.close()
    return {"task_id": task_id, **task.dict()}

@app.get("/tasks/", response_model=list[Task])
def read_tasks():
    db = get_db()
    cursor = db.cursor()
    query = "SELECT * FROM tasks"
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    db.close()
    tasks = []
    for row in results:
        tasks.append({
            "task_id": row[0],
            "title": row[1],
            "description": row[2],
            "due_date": row[3],
            "status": row[4],
            "created_at": row[5],
            "updated_at": row[6]
        })
    return tasks

@app.get("/tasks/{task_id}", response_model=Task)
def read_task(task_id: int):
    db = get_db()
    cursor = db.cursor()
    query = "SELECT * FROM tasks WHERE task_id = %s"
    cursor.execute(query, (task_id,))
    result = cursor.fetchone()
    cursor.close()
    db.close()
    if result is None:
        raise HTTPException(status_code=404, detail="Task not found")
    return {
        "task_id": result[0],
        "title": result[1],
        "description": result[2],
        "due_date": result[3],
        "status": result[4],
        "created_at": result[5],
        "updated_at": result[6]
    }






@app.put("/tasks/{task_id}", response_model=Task)
def update_task(task_id: int, task: TaskUpdate):
    db = get_db()
    cursor = db.cursor()
    query = "UPDATE tasks SET title=%s, description=%s, due_date=%s, status=%s WHERE task_id=%s"
    values = (task.title, task.description, task.due_date, task.status, task_id)
    cursor.execute(query, values)
    db.commit()
    cursor.close()
    db.close()
    return {"task_id": task_id, **task.dict()}

@app.delete("/tasks/{task_id}", status_code=204)
def delete_task(task_id: int):
    db = get_db()
    cursor = db.cursor()
    query = "DELETE FROM tasks WHERE task_id=%s"
    cursor.execute(query, (task_id,))
    db.commit()
    cursor.close()
    db.close()
    return {"message": f"Task with id {task_id} deleted successfully"}
