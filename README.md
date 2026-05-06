# kod.mob
expense_tracker/
├── expense_tracker.py   # Главный файл приложения
├── expenses.json        # Файл с данными (создаётся автоматически)
├── .gitignore
└── README.md

import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime
import json
import os

DATA_FILE = "expenses.json"


class ExpenseTracker:
    def __init__(self, root):
        self.root = root
        self.root.title("Expense Tracker - Личный трекер расходов")
        self.root.geometry("800x500")
        self.root.resizable(True, True)

        # Данные: список расходов
        self.expenses = []
        self.load_data()

        # Создание интерфейса
        self.create_widgets()
        self.update_table()

    def create_widgets(self):
        # Рамка для ввода данных
        input_frame = ttk.LabelFrame(self.root, text="Добавить расход", padding=10)
        input_frame.pack(fill="x", padx=10, pady=5)

        # Поле "Сумма"
        ttk.Label(input_frame, text="Сумма:").grid(row=0, column=0, padx=5, pady=5, sticky="e")
        self.amount_entry = ttk.Entry(input_frame, width=20)
        self.amount_entry.grid(row=0, column=1, padx=5, pady=5)

        # Поле "Категория"
        ttk.Label(input_frame, text="Категория:").grid(row=0, column=2, padx=5, pady=5, sticky="e")
        self.category_var = tk.StringVar()
        categories = ["Еда", "Транспорт", "Развлечения", "Здоровье", "Одежда", "Другое"]
        self.category_combo = ttk.Combobox(input_frame, textvariable=self.category_var, values=categories, width=18)
        self.category_combo.grid(row=0, column=3, padx=5, pady=5)
        self.category_combo.set("Еда")

        # Поле "Дата"
        ttk.Label(input_frame, text="Дата (ГГГГ-ММ-ДД):").grid(row=0, column=4, padx=5, pady=5, sticky="e")
        self.date_entry = ttk.Entry(input_frame, width=12)
        self.date_entry.grid(row=0, column=5, padx=5, pady=5)
        self.date_entry.insert(0, datetime.now().strftime("%Y-%m-%d"))

        # Кнопка добавления
        self.add_btn = ttk.Button(input_frame, text="➕ Добавить расход", command=self.add_expense)
        self.add_btn.grid(row=0, column=6, padx=10, pady=5)

        # Рамка для фильтров
        filter_frame = ttk.LabelFrame(self.root, text="Фильтры", padding=10)
        filter_frame.pack(fill="x", padx=10, pady=5)

        ttk.Label(filter_frame, text="Категория:").grid(row=0, column=0, padx=5, pady=5)
        self.filter_category_var = tk.StringVar()
        self.filter_category_combo = ttk.Combobox(filter_frame, textvariable=self.filter_category_var,
                                                  values=["Все"] + categories, width=18)
        self.filter_category_combo.grid(row=0, column=1, padx=5, pady=5)
        self.filter_category_combo.set("Все")

        ttk.Label(filter_frame, text="Дата с (ГГГГ-ММ-ДД):").grid(row=0, column=2, padx=5, pady=5)
        self.start_date_entry = ttk.Entry(filter_frame, width=12)
        self.start_date_entry.grid(row=0, column=3, padx=5, pady=5)

        ttk.Label(filter_frame, text="по:").grid(row=0, column=4, padx=5, pady=5)
        self.end_date_entry = ttk.Entry(filter_frame, width=12)
        self.end_date_entry.grid(row=0, column=5, padx=5, pady=5)

        self.filter_btn = ttk.Button(filter_frame, text="🔍 Применить фильтр", command=self.apply_filter)
        self.filter_btn.grid(row=0, column=6, padx=10, pady=5)

        self.clear_filter_btn = ttk.Button(filter_frame, text="❌ Сбросить фильтр", command=self.clear_filter)
        self.clear_filter_btn.grid(row=0, column=7, padx=5, pady=5)

        # Рамка для суммы за период
        sum_frame = ttk.Frame(self.root)
        sum_frame.pack(fill="x", padx=10, pady=5)

        self.calc_sum_btn = ttk.Button(sum_frame, text="💰 Подсчитать сумму за выбранный период",
                                       command=self.calculate_sum)
        self.calc_sum_btn.pack(side="left", padx=5)

        self.sum_label = ttk.Label(sum_frame, text="Сумма: 0.00 руб.", font=("Arial", 10, "bold"))
        self.sum_label.pack(side="left", padx=10)

        # Таблица для отображения расходов
        table_frame = ttk.Frame(self.root)
        table_frame.pack(fill="both", expand=True, padx=10, pady=10)

        columns = ("ID", "Сумма", "Категория", "Дата")
        self.tree = ttk.Treeview(table_frame, columns=columns, show="headings", height=15)

        self.tree.heading("ID", text="ID")
        self.tree.heading("Сумма", text="Сумма (руб.)")
        self.tree.heading("Категория", text="Категория")
        self.tree.heading("Дата", text="Дата")

        self.tree.column("ID", width=40, anchor="center")
        self.tree.column("Сумма", width=100, anchor="e")
        self.tree.column("Категория", width=120, anchor="w")
        self.tree.column("Дата", width=100, anchor="center")

        scrollbar = ttk.Scrollbar(table_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        # Кнопка удаления записи
        self.delete_btn = ttk.Button(self.root, text="🗑 Удалить выбранную запись", command=self.delete_expense)
        self.delete_btn.pack(pady=5)

    def add_expense(self):
        """Добавление нового расхода с проверками"""
        # Проверка суммы
        try:
            amount = float(self.amount_entry.get())
            if amount <= 0:
                messagebox.showerror("Ошибка", "Сумма должна быть положительным числом.")
                return
        except ValueError:
            messagebox.showerror("Ошибка", "Сумма должна быть числом.")
            return

        category = self.category_var.get().strip()
        date_str = self.date_entry.get().strip()

        # Проверка формата даты
        try:
            datetime.strptime(date_str, "%Y-%m-%d")
        except ValueError:
            messagebox.showerror("Ошибка", "Неверный формат даты. Используйте ГГГГ-ММ-ДД (например, 2025-03-30).")
            return

        # Создание нового расхода
        new_id = max([exp["id"] for exp in self.expenses], default=0) + 1
        new_expense = {
            "id": new_id,
            "amount": amount,
            "category": category,
            "date": date_str
        }
        self.expenses.append(new_expense)
        self.save_data()
        self.update_table()
        self.clear_input_fields()
        messagebox.showinfo("Успех", "Расход добавлен!")

    def delete_expense(self):
        """Удаление выбранного расхода"""
        selected_item = self.tree.selection()
        if not selected_item:
            messagebox.showwarning("Предупреждение", "Выберите запись для удаления.")
            return

        # Получаем ID записи из таблицы
        item = self.tree.item(selected_item[0])
        expense_id = int(item["values"][0])

        # Удаляем из списка
        self.expenses = [exp for exp in self.expenses if exp["id"] != expense_id]
        self.save_data()
        self.update_table()
        self.apply_filter()  # Переприменяем текущий фильтр
        messagebox.showinfo("Успех", "Запись удалена.")

    def update_table(self, filtered_expenses=None):
        """Обновление таблицы на основе отфильтрованных данных"""
        # Очищаем таблицу
        for row in self.tree.get_children():
            self.tree.delete(row)

        data = filtered_expenses if filtered_expenses is not None else self.expenses
        for exp in data:
            self.tree.insert("", "end", values=(exp["id"], f"{exp['amount']:.2f}", exp["category"], exp["date"]))

    def apply_filter(self):
        """Фильтрация по категории и диапазону дат"""
        filtered = self.expenses.copy()

        # Фильтр по категории
        category_filter = self.filter_category_var.get()
        if category_filter != "Все":
            filtered = [exp for exp in filtered if exp["category"] == category_filter]

        # Фильтр по дате
        start_date = self.start_date_entry.get().strip()
        end_date = self.end_date_entry.get().strip()

        if start_date:
            try:
                start_dt = datetime.strptime(start_date, "%Y-%m-%d")
                filtered = [exp for exp in filtered if
                            datetime.strptime(exp["date"], "%Y-%m-%d") >= start_dt]
            except ValueError:
                messagebox.showerror("Ошибка", "Неверный формат начальной даты. Используйте ГГГГ-ММ-ДД.")
                return

        if end_date:
            try:
                end_dt = datetime.strptime(end_date, "%Y-%m-%d")
                filtered = [exp for exp in filtered if
                            datetime.strptime(exp["date"], "%Y-%m-%d") <= end_dt]
            except ValueError:
                messagebox.showerror("Ошибка", "Неверный формат конечной даты. Используйте ГГГГ-ММ-ДД.")
                return

        self.update_table(filtered)

    def clear_filter(self):
        """Сброс всех фильтров"""
        self.filter_category_var.set("Все")
        self.start_date_entry.delete(0, tk.END)
        self.end_date_entry.delete(0, tk.END)
        self.update_table()
        self.sum_label.config(text="Сумма: 0.00 руб.")

    def calculate_sum(self):
        """Подсчёт суммы расходов за выбранный в фильтрах период"""
        filtered_expenses = self.expenses.copy()

        category_filter = self.filter_category_var.get()
        if category_filter != "Все":
            filtered_expenses = [exp for exp in filtered_expenses if exp["category"] == category_filter]

        start_date = self.start_date_entry.get().strip()
        end_date = self.end_date_entry.get().strip()

        if start_date:
            try:
                start_dt = datetime.strptime(start_date, "%Y-%m-%d")
                filtered_expenses = [exp for exp in filtered_expenses if
                                     datetime.strptime(exp["date"], "%Y-%m-%d") >= start_dt]
            except ValueError:
                messagebox.showerror("Ошибка", "Неверный формат начальной даты.")
                return

        if end_date:
            try:
                end_dt = datetime.strptime(end_date, "%Y-%m-%d")
                filtered_expenses = [exp for exp in filtered_expenses if
                                     datetime.strptime(exp["date"], "%Y-%m-%d") <= end_dt]
            except ValueError:
                messagebox.showerror("Ошибка", "Неверный формат конечной даты.")
                return

        total = sum(exp["amount"] for exp in filtered_expenses)
        self.sum_label.config(text=f"Сумма: {total:.2f} руб.")

        if filtered_expenses:
            messagebox.showinfo("Результат", f"Сумма расходов за выбранный период: {total:.2f} руб.")
        else:
            messagebox.showinfo("Результат", "Нет расходов за выбранный период.")

    def clear_input_fields(self):
        """Очистка полей ввода"""
        self.amount_entry.delete(0, tk.END)
        self.category_var.set("Еда")
        self.date_entry.delete(0, tk.END)
        self.date_entry.insert(0, datetime.now().strftime("%Y-%m-%d"))

    def save_data(self):
        """Сохранение данных в JSON"""
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump(self.expenses, f, ensure_ascii=False, indent=4)

    def load_data(self):
        """Загрузка данных из JSON"""
        if os.path.exists(DATA_FILE):
            try:
                with open(DATA_FILE, "r", encoding="utf-8") as f:
                    self.expenses = json.load(f)
            except (json.JSONDecodeError, FileNotFoundError):
                self.expenses = []
        else:
            self.expenses = []


if __name__ == "__main__":
    root = tk.Tk()
    app = ExpenseTracker(root)
    root.mainloop()

    # Python
__pycache__/
*.py[cod]
*.so
.Python

# Virtual environments
venv/
env/
ENV/

# IDE
.vscode/
.idea/
*.swp
*.swo

# Project specific
expenses.json
*.log
.DS_Store

git init
git add .
git commit -m "Initial commit: Expense Tracker app"
git remote add origin https://github.com/yourusername/expense-tracker.git
git push -u origin main

