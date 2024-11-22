import tkinter as tk
from tkinter import ttk, messagebox
import sqlite3
import random

# Database setup
conn = sqlite3.connect('library.db')
c = conn.cursor()

# Create table if not exists
c.execute('''CREATE TABLE IF NOT EXISTS books
             (id TEXT PRIMARY KEY,
              title TEXT,
              author TEXT,
              year INTEGER,
              status TEXT,
              available_quantity INTEGER,
              checked_out_quantity INTEGER)''')

# Function to generate a unique ID
def generate_unique_id():
    while True:
        new_id = str(random.randint(10000, 99999))
        c.execute("SELECT id FROM books WHERE id=?", (new_id,))
        if not c.fetchone():
            return new_id

# Insert initial data only if the table is empty
c.execute("SELECT COUNT(*) FROM books")
count = c.fetchone()[0]
if count == 0:
    initial_books = [
        ('Astrophysics for People in a Hurry', 'Neil deGrasse Tyson', 2017),
        ('Death by Black Hole', 'Neil deGrasse Tyson', 2007),
        ('Letters from an Astrophysicist', 'Neil deGrasse Tyson', 2019),
        ('Undeniable: Evolution and the Science of Creation', 'Bill Nye', 2014),
        ('Unstoppable: Harnessing Science to Change the World', 'Bill Nye', 2015)
    ]
    
    for title, author, year in initial_books:
        book_id = generate_unique_id()
        quantity = random.randint(1, 5)
        c.execute('''INSERT INTO books (id, title, author, year, status, available_quantity, checked_out_quantity) 
                   VALUES (?, ?, ?, ?, 'Available', ?, 0)''', 
                  (book_id, title, author, year, quantity))
    conn.commit()

# GUI setup
root = tk.Tk()
root.title("Library Management System")
root.minsize(800, 600)
style = ttk.Style(root)
style.theme_use('clam')

def ask_remove_option():
    """Create a custom dialog to choose removal option."""
    dialog = tk.Toplevel(root)
    dialog.title("Remove Book")
    dialog.grab_set()  # Make the dialog modal
    dialog.resizable(False, False)

    ttk.Label(dialog, text="Choose removal option:").pack(padx=20, pady=10)

    button_frame = ttk.Frame(dialog)
    button_frame.pack(pady=10)

    choice = tk.StringVar(value="None")  # Default choice

    def select_all():
        choice.set("All")
        dialog.destroy()

    def select_one():
        choice.set("One")
        dialog.destroy()

    def select_none():
        choice.set("None")
        dialog.destroy()

    ttk.Button(button_frame, text="All", command=select_all, width=10).grid(row=0, column=0, padx=5)
    ttk.Button(button_frame, text="One", command=select_one, width=10).grid(row=0, column=1, padx=5)
    ttk.Button(button_frame, text="None", command=select_none, width=10).grid(row=0, column=2, padx=5)

    dialog.wait_window()  # Wait for the dialog to be closed
    return choice.get()

def on_click(event):
    """
    Event handler to unselect Treeviews when clicking outside of interactive widgets.
    """
    widget = event.widget
    # Updated list to include 'TEntry' for ttk.Entry widgets
    interactive_widget_classes = ('TButton', 'TEntry', 'Treeview', 'TCombobox', 'Text', 'Radiobutton', 'Checkbutton')

    # Traverse the widget hierarchy to check if the clicked widget is child of an interactive widget
    is_interactive = False
    parent = widget
    while parent is not None:
        if parent.winfo_class() in interactive_widget_classes:
            is_interactive = True
            break
        try:
            parent = parent.master
        except AttributeError:
            break

    if not is_interactive:
        tree.selection_remove(tree.selection())
        checked_out_tree.selection_remove(checked_out_tree.selection())

def add_book():
    selected = tree.selection()
    if selected:
        values = tree.item(selected[0])['values']
        book_id = values[0]
        title = values[1]
        
        # Confirm adding a copy
        confirm = messagebox.askyesno("Add Copy", f"Add another copy of '{title}'?")
        if confirm:
            try:
                c.execute('''UPDATE books 
                            SET available_quantity = available_quantity + 1
                            WHERE id = ?''', (book_id,))
                conn.commit()
                messagebox.showinfo("Success", f"Another copy of '{title}' added successfully.")
                display_books()
                display_checked_out_books()
            except Exception as e:
                messagebox.showerror("Error", f"Failed to add copy: {e}")
    else:
        # Proceed to add a new book
        title = title_entry.get().strip()
        author = author_entry.get().strip()
        year = year_entry.get().strip()
        if title and author and year:
            if not year.isdigit():
                messagebox.showwarning("Invalid Input", "Year must be a number.")
                return
            # Check if the book already exists
            c.execute('''SELECT id FROM books 
                         WHERE title = ? AND author = ? AND year = ?''', 
                      (title, author, int(year)))
            existing_book = c.fetchone()
            if existing_book:
                # Prompt to add a copy instead
                add_copy = messagebox.askyesno("Book Exists", f"A book titled '{title}' by {author} ({year}) already exists. Do you want to add a copy instead?")
                if add_copy:
                    try:
                        c.execute('''UPDATE books 
                                     SET available_quantity = available_quantity + 1
                                     WHERE id = ?''', (existing_book[0],))
                        conn.commit()
                        messagebox.showinfo("Success", f"Another copy of '{title}' added successfully.")
                        clear_entries()
                        display_books()
                        display_checked_out_books()
                    except Exception as e:
                        messagebox.showerror("Error", f"Failed to add copy: {e}")
                    return  # Exit the function after adding a copy

            # If the book does not exist, insert as a new book
            book_id = generate_unique_id()
            try:
                c.execute('''INSERT INTO books 
                            (id, title, author, year, status, available_quantity, checked_out_quantity)
                            VALUES (?, ?, ?, ?, 'Available', 1, 0)''',
                         (book_id, title, author, int(year)))
                conn.commit()
                messagebox.showinfo("Success", f"Book '{title}' added successfully.")
                clear_entries()
                display_books()
                display_checked_out_books()
                # Clear any lingering selections
                tree.selection_remove(tree.selection())
                checked_out_tree.selection_remove(checked_out_tree.selection())
            except sqlite3.IntegrityError:
                messagebox.showwarning("Error", "Book with the same ID already exists.")
            except Exception as e:
                messagebox.showerror("Error", f"Failed to add book: {e}")
        else:
            messagebox.showwarning("Input Required", "Please fill all the fields to add a new book.")

def remove_book():
    selected = tree.focus()
    if selected:
        values = tree.item(selected)['values']
        book_id = values[0]
        title = values[1]
        
        choice = ask_remove_option()
        
        if choice == "All":
            confirm = messagebox.askyesno("Confirm Deletion", f"Are you sure you want to delete all copies of '{title}'?")
            if confirm:
                try:
                    c.execute("DELETE FROM books WHERE id=?", (book_id,))
                    conn.commit()
                    messagebox.showinfo("Success", f"All copies of '{title}' have been removed.")
                    display_books()
                    display_checked_out_books()
                except Exception as e:
                    messagebox.showerror("Error", f"Failed to remove book: {e}")
        
        elif choice == "One":
            available = values[5]
            if available > 0:
                try:
                    c.execute('''UPDATE books 
                                SET available_quantity = available_quantity - 1
                                WHERE id = ?''', (book_id,))
                    conn.commit()
                    messagebox.showinfo("Success", f"One copy of '{title}' has been removed.")
                    display_books()
                    display_checked_out_books()
                except Exception as e:
                    messagebox.showerror("Error", f"Failed to remove one copy: {e}")
            else:
                messagebox.showwarning("Unavailable", "No available copies to remove.")
        
        elif choice == "None":
            # Do nothing
            pass
        else:
            # User closed the dialog or unexpected choice
            pass
    else:
        messagebox.showwarning("Selection Required", "Please select a book to remove.")

def checkout_book():
    print("Checkout Book Function Called")  # Debugging Statement
    selected = tree.selection()
    if selected:
        values = tree.item(selected[0])['values']
        book_id = values[0]
        title = values[1]
        available = values[5]  # Correctly mapped to available_quantity
        
        print(f"Book ID: {book_id}, Title: {title}, Available: {available}")  # Debugging Statement
        
        if available > 0:
            try:
                c.execute('''UPDATE books 
                            SET available_quantity = available_quantity - 1,
                                checked_out_quantity = checked_out_quantity + 1
                            WHERE id = ?''', (book_id,))
                conn.commit()
                messagebox.showinfo("Success", f"Book '{title}' checked out successfully.")
                display_books()
                display_checked_out_books()
            except Exception as e:
                messagebox.showerror("Error", f"Failed to checkout book: {e}")
        else:
            messagebox.showwarning("Unavailable", "No copies available for checkout.")
    else:
        messagebox.showwarning("Selection Required", "Please select a book to checkout.")

def return_book():
    print("Return Book Function Called")  # Debugging Statement
    selected = checked_out_tree.selection()
    if selected:
        values = checked_out_tree.item(selected[0])['values']
        book_id = values[0]
        title = values[1]
        checked_out = values[5]  # Correctly mapped to checked_out_quantity
        
        print(f"Book ID: {book_id}, Title: {title}, Checked Out: {checked_out}")  # Debugging Statement
        
        if checked_out > 0:
            try:
                c.execute('''UPDATE books 
                            SET available_quantity = available_quantity + 1,
                                checked_out_quantity = checked_out_quantity - 1
                            WHERE id = ?''', (book_id,))
                conn.commit()
                messagebox.showinfo("Success", f"Book '{title}' returned successfully.")
                display_books()
                display_checked_out_books()
            except Exception as e:
                messagebox.showerror("Error", f"Failed to return book: {e}")
        else:
            messagebox.showwarning("Error", "No copies are currently checked out.")
    else:
        messagebox.showwarning("Selection Required", "Please select a book to return.")

def search_books():
    search_term = search_entry.get().strip()
    for item in tree.get_children():
        tree.delete(item)
    for item in checked_out_tree.get_children():
        checked_out_tree.delete(item)
    if search_term:
        query = '''SELECT * FROM books 
                   WHERE id LIKE ? OR title LIKE ? OR author LIKE ? OR year LIKE ?'''
        terms = ['%' + search_term + '%'] * 4
        try:
            for row in c.execute(query, terms):
                tree.insert('', tk.END, values=row)
                if row[6] > 0:
                    # Correctly map 'Checked Out' to checked_out_quantity
                    checked_out_tree.insert('', tk.END, values=(row[0], row[1], row[2], row[3], row[4], row[6]))
        except Exception as e:
            messagebox.showerror("Error", f"Failed to search books: {e}")
    else:
        display_books()
        display_checked_out_books()

def display_books():
    for item in tree.get_children():
        tree.delete(item)
    try:
        for row in c.execute("SELECT * FROM books WHERE available_quantity > 0"):
            tree.insert('', tk.END, values=row)
    except Exception as e:
        messagebox.showerror("Error", f"Failed to display books: {e}")

def display_checked_out_books():
    for item in checked_out_tree.get_children():
        checked_out_tree.delete(item)
    try:
        for row in c.execute("SELECT * FROM books WHERE checked_out_quantity > 0"):
            # Map the correct columns, ensuring 'Checked Out' maps to checked_out_quantity (row[6])
            checked_out_tree.insert('', tk.END, values=(row[0], row[1], row[2], row[3], row[4], row[6]))
    except Exception as e:
        messagebox.showerror("Error", f"Failed to display checked out books: {e}")

def clear_entries():
    title_entry.delete(0, tk.END)
    author_entry.delete(0, tk.END)
    year_entry.delete(0, tk.END)
    search_entry.delete(0, tk.END)

def on_closing():
    try:
        conn.close()
    except:
        pass
    root.destroy()

def update_book():
    selected = tree.selection()
    if selected:
        values = tree.item(selected[0])['values']
        book_id = values[0]
        
        title = title_entry.get().strip()
        author = author_entry.get().strip()
        year = year_entry.get().strip()
        
        if title and author and year:
            if not year.isdigit():
                messagebox.showwarning("Invalid Input", "Year must be a number.")
                return
            try:
                c.execute('''UPDATE books 
                             SET title = ?, author = ?, year = ?
                             WHERE id = ?''', (title, author, int(year), book_id))
                conn.commit()
                messagebox.showinfo("Success", f"Book '{title}' updated successfully.")
                clear_entries()
                display_books()
                display_checked_out_books()
                tree.selection_remove(tree.selection())
                checked_out_tree.selection_remove(checked_out_tree.selection())
            except Exception as e:
                messagebox.showerror("Error", f"Failed to update book: {e}")
        else:
            messagebox.showwarning("Input Required", "Please fill all the fields to update the book.")
    else:
        messagebox.showwarning("Selection Required", "Please select a book to update.")

# Optionally, populate the entry fields when a book is selected
def on_tree_select(event):
    selected = tree.selection()
    if selected:
        values = tree.item(selected[0])['values']
        title_entry.delete(0, tk.END)
        title_entry.insert(0, values[1])
        author_entry.delete(0, tk.END)
        author_entry.insert(0, values[2])
        year_entry.delete(0, tk.END)
        year_entry.insert(0, values[3])

# Create main frames
input_frame = ttk.Frame(root, padding="10 10 10 10")
input_frame.grid(row=0, column=0, columnspan=2, sticky='ew')

button_frame = ttk.Frame(root, padding="10 10 10 10")
button_frame.grid(row=1, column=0, columnspan=2, sticky='ew')

search_frame = ttk.Frame(root, padding="10 10 10 10")
search_frame.grid(row=2, column=0, columnspan=2, sticky='ew')

tree_frame = ttk.Frame(root, padding="10 10 10 10")
tree_frame.grid(row=3, column=0, columnspan=2, sticky='nsew')

# Configure grid weights
root.grid_rowconfigure(3, weight=1)
root.grid_columnconfigure(0, weight=1)
root.grid_columnconfigure(1, weight=1)

# Input Fields
ttk.Label(input_frame, text="Title:").grid(row=0, column=0, padx=3, pady=2, sticky='e')
ttk.Label(input_frame, text="Author:").grid(row=1, column=0, padx=3, pady=2, sticky='e')
ttk.Label(input_frame, text="Year:").grid(row=2, column=0, padx=3, pady=2, sticky='e')

title_entry = ttk.Entry(input_frame)
author_entry = ttk.Entry(input_frame)
year_entry = ttk.Entry(input_frame)

title_entry.grid(row=0, column=1, padx=3, pady=2, sticky='w')
author_entry.grid(row=1, column=1, padx=3, pady=2, sticky='w')
year_entry.grid(row=2, column=1, padx=3, pady=2, sticky='w')

# Buttons with uniform size
button_width = 15

add_button = ttk.Button(button_frame, text="Add Book", command=add_book, width=button_width)
remove_button = ttk.Button(button_frame, text="Remove Book", command=remove_book, width=button_width)
checkout_button = ttk.Button(button_frame, text="Checkout Book", command=checkout_book, width=button_width)
return_button = ttk.Button(button_frame, text="Return Book", command=return_book, width=button_width)
update_button = ttk.Button(button_frame, text="Update Book", command=update_book, width=button_width)

add_button.grid(row=0, column=0, padx=5, pady=5)
remove_button.grid(row=0, column=1, padx=5, pady=5)
checkout_button.grid(row=0, column=2, padx=5, pady=5)
return_button.grid(row=0, column=3, padx=5, pady=5)
update_button.grid(row=0, column=4, padx=5, pady=5)

# Centered Search Section (Search label removed)
search_entry = ttk.Entry(search_frame, width=40)
search_entry.pack(side="left", padx=(0, 5))
ttk.Button(search_frame, text="Search", command=search_books, width=button_width).pack(side="left")

# Available Books Section
ttk.Label(tree_frame, text="Available Books").pack(anchor='w')
available_columns = ('ID', 'Title', 'Author', 'Year', 'Status', 'Available')
tree = ttk.Treeview(tree_frame, columns=available_columns, show='headings')
for col in available_columns:
    tree.heading(col, text=col)
    if col in ('Title', 'Author'):
        tree.column(col, width=300)
    elif col == 'ID':
        tree.column(col, width=70)
    elif col == 'Year':
        tree.column(col, width=70)
    else:
        tree.column(col, width=100)
tree.pack(fill='both', expand=True)

# Checked Out Books Section
ttk.Label(root, text="Checked Out Books").grid(row=4, column=0, columnspan=2, pady=5)
checked_columns = ('ID', 'Title', 'Author', 'Year', 'Status', 'Checked Out')
checked_out_tree = ttk.Treeview(root, columns=checked_columns, show='headings')
for col in checked_columns:
    checked_out_tree.heading(col, text=col)
    if col in ('Title', 'Author'):
        checked_out_tree.column(col, width=300)
    elif col == 'ID':
        checked_out_tree.column(col, width=70)
    elif col == 'Year':
        checked_out_tree.column(col, width=70)
    else:
        checked_out_tree.column(col, width=100)
checked_out_tree.grid(row=5, column=0, columnspan=2, padx=5, pady=5, sticky='nsew')

# Configure grid weights for tree_frame and root
tree_frame.grid_rowconfigure(0, weight=1)
tree_frame.grid_columnconfigure(0, weight=1)
root.grid_rowconfigure(5, weight=1)

# Initialize display
display_books()
display_checked_out_books()

# Bind the global click event to the on_click handler
root.bind("<Button-1>", on_click)

# Bind the selection event to populate entries
tree.bind("<<TreeviewSelect>>", on_tree_select)

# Handle window closing to ensure the database connection is closed
root.protocol("WM_DELETE_WINDOW", on_closing)

root.mainloop()