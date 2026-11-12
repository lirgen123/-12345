import tkinter as tk
from tkinter import ttk, messagebox
import sqlite3
from datetime import datetime

# --- РАБОТА С БАЗОЙ ДАННЫХ ---
def init_db():
    conn = sqlite3.connect('warehouse.db')
    cursor = conn.cursor()
    # Создание таблицы товаров
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS products (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            category TEXT,
            price REAL NOT NULL,
            quantity INTEGER NOT NULL
        )
    ''')
    # Проверка, есть ли уже данные. Если нет - добавляем 10 тестовых (по заданию)
    cursor.execute('SELECT COUNT(*) FROM products')
    if cursor.fetchone()[0] == 0:
        test_data = [
            ('Гвозди 100мм', 'Крепеж', 5.5, 1000),
            ('Молоток', 'Инструмент', 1500.0, 20),
            ('Шуруповерт', 'Электроинструмент', 4500.0, 10),
            ('Дрель', 'Электроинструмент', 3200.0, 5),
            ('Изолента', 'Расходники', 50.0, 200),
            ('Перчатки', 'СИЗ', 120.0, 500),
            ('Каска', 'СИЗ', 800.0, 30),
            ('Рулетка 5м', 'Инструмент', 450.0, 40),
            ('Уровень', 'Инструмент', 1200.0, 15),
            ('Набор отверток', 'Инструмент', 900.0, 25)
        ]
        cursor.executemany('INSERT INTO products (name, category, price, quantity) VALUES (?, ?, ?, ?)', test_data)
        conn.commit()
    conn.close()

# --- ГЛАВНОЕ ОКНО ---
class WarehouseApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Информационная система: Склад")
        self.root.geometry("800x600")
        
        # Верхняя панель (Поиск)
        search_frame = tk.Frame(root)
        search_frame.pack(pady=10)
        tk.Label(search_frame, text="Поиск по названию:").pack(side=tk.LEFT, padx=5)
        self.search_entry = tk.Entry(search_frame)
        self.search_entry.pack(side=tk.LEFT, padx=5)
        tk.Button(search_frame, text="Найти", command=self.load_data).pack(side=tk.LEFT, padx=5)
        tk.Button(search_frame, text="Сброс", command=self.reset_search).pack(side=tk.LEFT, padx=5)

        # Таблица (Treeview)
        self.tree = ttk.Treeview(root, columns=("ID", "Название", "Категория", "Цена", "Кол-во"), show="headings")
        self.tree.heading("ID", text="ID")
        self.tree.heading("Название", text="Название")
        self.tree.heading("Категория", text="Категория")
        self.tree.heading("Цена", text="Цена (руб)")
        self.tree.heading("Кол-во", text="Количество")
        
        self.tree.column("ID", width=40, anchor=tk.CENTER)
        self.tree.column("Название", width=200)
        self.tree.column("Категория", width=150)
        self.tree.column("Цена", width=100, anchor=tk.CENTER)
        self.tree.column("Кол-во", width=100, anchor=tk.CENTER)
        self.tree.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

        # Панель кнопок
        btn_frame = tk.Frame(root)
        btn_frame.pack(pady=10)
        
        tk.Button(btn_frame, text="Добавить", width=15, bg="#a8e6cf", command=self.add_item).grid(row=0, column=0, padx=5)
        tk.Button(btn_frame, text="Редактировать", width=15, bg="#ffd3b6", command=self.edit_item).grid(row=0, column=1, padx=5)
        tk.Button(btn_frame, text="Удалить", width=15, bg="#ffaaa5", command=self.delete_item).grid(row=0, column=2, padx=5)
        tk.Button(btn_frame, text="Приход (+)", width=15, command=lambda: self.change_qty(1)).grid(row=1, column=0, padx=5, pady=5)
        tk.Button(btn_frame, text="Расход (-)", width=15, command=lambda: self.change_qty(-1)).grid(row=1, column=1, padx=5, pady=5)
        tk.Button(btn_frame, text="Обновить", width=15, command=self.load_data).grid(row=1, column=2, padx=5, pady=5)

        self.load_data()

    def load_data(self):
        # Очистка таблицы
        for i in self.tree.get_children():
            self.tree.delete(i)
        
        search_term = self.search_entry.get()
        conn = sqlite3.connect('warehouse.db')
        cursor = conn.cursor()
        
        if search_term:
            cursor.execute("SELECT * FROM products WHERE name LIKE ?", ('%' + search_term + '%',))
        else:
            cursor.execute("SELECT * FROM products")
            
        rows = cursor.fetchall()
        for row in rows:
            self.tree.insert("", tk.END, values=row)
        conn.close()

    def reset_search(self):
        self.search_entry.delete(0, tk.END)
        self.load_data()

    # --- ОКНО ДОБАВЛЕНИЯ / РЕДАКТИРОВАНИЯ ---
    def open_form(self, title, item_data=None):
        form = tk.Toplevel(self.root)
        form.title(title)
        form.geometry("300x250")
        form.grab_set() # Блокирует главное окно

        tk.Label(form, text="Название:").pack(pady=5)
        name_entry = tk.Entry(form)
        name_entry.pack()

        tk.Label(form, text="Категория:").pack(pady=5)
        cat_entry = tk.Entry(form)
        cat_entry.pack()

        tk.Label(form, text="Цена:").pack(pady=5)
        price_entry = tk.Entry(form)
        price_entry.pack()

        tk.Label(form, text="Количество:").pack(pady=5)
        qty_entry = tk.Entry(form)
        qty_entry.pack()

        # Если переданы данные (режим редактирования), заполняем поля
        if item_data:
            name_entry.insert(0, item_data[1])
            cat_entry.insert(0, item_data[2])
            price_entry.insert(0, item_data[3])
            qty_entry.insert(0, item_data[4])

        def save():
            name = name_entry.get().strip()
            category = cat_entry.get().strip()
            
            # --- ОБРАБОТКА ОШИБОК (по заданию) ---
            if not name:
                messagebox.showerror("Ошибка", "Название не может быть пустым!")
                return
            try:
                price = float(price_entry.get())
                quantity = int(qty_entry.get())
                if price < 0 or quantity < 0:
                    raise ValueError
            except ValueError:
                messagebox.showerror("Ошибка ввода", "Цена и количество должны быть положительными числами!")
                return

            conn = sqlite3.connect('warehouse.db')
            cursor = conn.cursor()
            
            if item_data: # UPDATE
                cursor.execute("UPDATE products SET name=?, category=?, price=?, quantity=? WHERE id=?", 
                               (name, category, price, quantity, item_data[0]))
            else: # INSERT
                cursor.execute("INSERT INTO products (name, category, price, quantity) VALUES (?, ?, ?, ?)", 
                               (name, category, price, quantity))
            
            conn.commit()
            conn.close()
            form.destroy()
            self.load_data()

        tk.Button(form, text="Сохранить", bg="#a8e6cf", command=save).pack(pady=15)

    def add_item(self):
        self.open_form("Добавление товара")

    def edit_item(self):
        selected = self.tree.selection()
        if not selected:
            messagebox.showwarning("Внимание", "Выберите товар для редактирования!")
            return
        item_data = self.tree.item(selected[0])['values']
        self.open_form("Редактирование товара", item_data)

    def delete_item(self):
        selected = self.tree.selection()
        if not selected:
            messagebox.showwarning("Внимание", "Выберите товар для удаления!")
            return
        
        if messagebox.askyesno("Подтверждение", "Вы уверены, что хотите удалить этот товар?"):
            item_id = self.tree.item(selected[0])['values'][0]
            conn = sqlite3.connect('warehouse.db')
            cursor = conn.cursor()
            cursor.execute("DELETE FROM products WHERE id=?", (item_id,))
            conn.commit()
            conn.close()
            self.load_data()

    # --- ПРИХОД / РАСХОД ---
    def change_qty(self, sign):
        selected = self.tree.selection()
        if not selected:
            messagebox.showwarning("Внимание", "Выберите товар!")
            return
        
        item_data = self.tree.item(selected[0])['values']
        item_id, current_qty = item_data[0], item_data[4]

        # Диалоговое окно для ввода количества
        qty_window = tk.Toplevel(self.root)
        qty_window.title("Изменить количество")
        qty_window.geometry("250x150")
        qty_window.grab_set()

        action = "Приход" if sign == 1 else "Расход"
        tk.Label(qty_window, text=f"{action} для: {item_data[1]}").pack(pady=10)
        tk.Label(qty_window, text="Введите количество:").pack()
        qty_entry = tk.Entry(qty_window)
        qty_entry.pack(pady=5)

        def apply():
            try:
                change = int(qty_entry.get())
                if change <= 0:
                    raise ValueError
            except ValueError:
                messagebox.showerror("Ошибка", "Введите целое положительное число!")
                return
            
            new_qty = current_qty + (change * sign)
            if new_qty < 0:
                messagebox.showerror("Ошибка", "Недостаточно товара на складе для расхода!")
                return

            conn = sqlite3.connect('warehouse.db')
            cursor = conn.cursor()
            cursor.execute("UPDATE products SET quantity=? WHERE id=?", (new_qty, item_id))
            conn.commit()
            conn.close()
            qty_window.destroy()
            self.load_data()

        tk.Button(qty_window, text="Применить", command=apply).pack(pady=10)

# --- АВТОРИЗАЦИЯ ---
def login():
    login_win = tk.Tk()
    login_win.title("Авторизация в системе Склад")
    login_win.geometry("300x200")

    tk.Label(login_win, text="Логин:").pack(pady=5)
    user_entry = tk.Entry(login_win)
    user_entry.pack()
    user_entry.insert(0, "admin") # Для удобства сдачи

    tk.Label(login_win, text="Пароль:").pack(pady=5)
    pass_entry = tk.Entry(login_win, show="*")
    pass_entry.pack()
    pass_entry.insert(0, "123") # Для удобства сдачи

    def check_login():
        # Простейшая проверка (по заданию "хоть бы проступ" логин/пароль)
        if user_entry.get() == "admin" and pass_entry.get() == "123":
            login_win.destroy()
            init_db()
            root = tk.Tk()
            app = WarehouseApp(root)
            root.mainloop()
        else:
            messagebox.showerror("Ошибка", "Неверный логин или пароль!")

    tk.Button(login_win, text="Войти", bg="#a8e6cf", command=check_login).pack(pady=15)
    login_win.mainloop()

if __name__ == "__main__":
    login()
