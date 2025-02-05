# alarm_clock
import sqlite3
import threading
import tkinter as tk
import datetime
import pygame
from tkinter import messagebox

# Database helper functions
def create_connection():
    return sqlite3.connect("clock_app.db")

def setup_database():
    conn = create_connection()
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS alert_clock (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            alarm_time TEXT,
            timestamp INTEGER,
            active INTEGER
        )
    """)
    conn.commit()
    conn.close()

def add_alarm(alarm_time, timestamp):
    conn = create_connection()
    cur = conn.cursor()
    cur.execute("INSERT INTO alert_clock (alarm_time, timestamp, active) VALUES (?, ?, 1)", (alarm_time, timestamp))
    conn.commit()
    conn.close()

def get_active_alarms():
    conn = create_connection()
    cur = conn.cursor()
    cur.execute("SELECT * FROM alert_clock WHERE active = 1 ORDER BY timestamp")
    alarms = cur.fetchall()
    conn.close()
    return alarms

def deactivate_alarm(alarm_id):
    conn = create_connection()
    cur = conn.cursor()
    cur.execute("UPDATE alert_clock SET active = 0 WHERE id = ?", (alarm_id,))
    conn.commit()
    conn.close()

# GUI Application
class AlarmApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Alarm Clock")
        self.root.geometry("400x300")
        
        tk.Label(root, text="Set New Alarm (HH:MM:SS)").pack()
        
        self.alarm_entry = tk.Entry(root)
        self.alarm_entry.pack()

        tk.Button(root, text="Set Alarm", command=self.set_alarm).pack()

        self.alarm_list_frame = tk.Frame(root)
        self.alarm_list_frame.pack(fill="both", expand=True)

        self.update_alarm_list()

        threading.Thread(target=self.check_alarms, daemon=True).start()

    def set_alarm(self):
        alarm_time = self.alarm_entry.get()
        try:
            alarm_datetime = datetime.datetime.strptime(alarm_time, "%H:%M:%S")
            today = datetime.datetime.today()
            alarm_datetime = today.replace(hour=alarm_datetime.hour, minute=alarm_datetime.minute, second=alarm_datetime.second)
            
            timestamp = int(alarm_datetime.timestamp())
            add_alarm(alarm_time, timestamp)
            
            self.update_alarm_list()
            messagebox.showinfo("Success", f"Alarm set for {alarm_time}")
        except ValueError:
            messagebox.showerror("Error", "Invalid time format. Use HH:MM:SS.")

    def update_alarm_list(self):
        for widget in self.alarm_list_frame.winfo_children():
            widget.destroy()

        alarms = get_active_alarms()
        for alarm in alarms:
            alarm_id, time, _, _ = alarm
            frame = tk.Frame(self.alarm_list_frame)
            frame.pack(fill="x", padx=5, pady=2)

            tk.Label(frame, text=f"⏰ {time}").pack(side="left")
            tk.Button(frame, text="Delete", command=lambda id=alarm_id: self.delete_alarm(id)).pack(side="right")

    def delete_alarm(self, alarm_id):
        deactivate_alarm(alarm_id)
        self.update_alarm_list()

    def check_alarms(self):
        pygame.mixer.init()
        while True:
            now = int(datetime.datetime.now().timestamp())
            alarms = get_active_alarms()

            for alarm in alarms:
                alarm_id, time, timestamp, active = alarm
                if active and timestamp <= now:
                    pygame.mixer.Sound("xipnitiri.wav").play()
                    messagebox.showinfo("Alarm!", f"Time to wake up! ⏰ {time}")
                    deactivate_alarm(alarm_id)
                    self.update_alarm_list()

setup_database()

root = tk.Tk()
app = AlarmApp(root)
root.mainloop()
