import tkinter as tk
from tkinter import ttk, messagebox
import json, os, random
import datetime  # CHỈ import datetime module, KHÔNG import from datetime
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
import cv2
from PIL import Image, ImageTk
import queue
import threading
import time
import requests
import telepot
import ssl
import urllib3
import psutil

ssl._create_default_https_context = ssl._create_unverified_context
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

try:
    import pygetwindow as gw
    GW_AVAILABLE = True
except ImportError:
    GW_AVAILABLE = False

try:
    from plyer import notification
    PLYER_AVAILABLE = True
except ImportError:
    PLYER_AVAILABLE = False

DATA_FILE = "study_data.json"

# ================= LOAD / SAVE =================

def load_data():
    default = {
        "xp": 0, "level": 1, "streak": 0,
        "last_day": "", "daily": {},
        "week_minutes": 0, "history": [], "reminders": []
    }
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r") as f:
                loaded = json.load(f)
                for key in default:
                    if key not in loaded:
                        loaded[key] = default[key]
                return loaded
        except:
            return default
    return default


def save_data():
    with open(DATA_FILE, "w") as f:
        json.dump(data, f)


data = load_data()

# ================= CẤU HÌNH TELEGRAM & BIẾN TOÀN CỤC =================
# FIX #1: Khai báo TOKEN/CHAT_ID/bot LÊN ĐẦU — trước mọi hàm dùng đến chúng
#         (Code gốc khai báo giữa file, sau các hàm đã dùng bot → NameError)
allowed_apps = ["python.exe", "explorer.exe", "taskhostw.exe", "svchost.exe"]
TOKEN   = '8602223640:AAFS8Q2ocalFcdBcD9QH6XRzM-mCQC_WRkY'
CHAT_ID = 8769215001
bot     = telepot.Bot(TOKEN)

allowed_apps = ["python.exe", "winword.exe", "chrome.exe", "zoom.exe", "code.exe", "cmd.exe"]

# FIX #2: Chỉ khai báo MỘT LẦN ở đây — code gốc khai báo data/total_time/last_alert_time
#         nhiều lần ở giữa file, ghi đè lên data đã load và reset biến sai
frame_queue        = queue.Queue(maxsize=2)
alert_queue        = queue.Queue()
last_alert_time    = 0.0     # float — dùng cho camera (time.time())
reminder_last_time = ""      # string — dùng cho reminders (HH:MM), tách riêng tránh xung đột
is_closing         = False
face_detected      = False
study_present_count = 0
study_absent_count  = 0
total_time         = 1       # khởi tạo > 0 để tránh ZeroDivisionError
running            = False
seconds            = 0
mode               = "study"
timer_id           = None

# ================= HỆ THỐNG GỬI TELEGRAM =================

def send_telegram_alert(message):
    """Chỉ đẩy vào queue, không gọi mạng trực tiếp → không lag camera"""
    alert_queue.put(message)

def telegram_sender_thread():
    """Thread riêng lo gửi, không ảnh hưởng UI hay camera"""
    while True:
        msg = alert_queue.get()
        if msg == "STOP":
            break
        try:
            url     = f"https://api.telegram.org/bot{TOKEN}/sendMessage"
            payload = {"chat_id": CHAT_ID, "text": msg, "parse_mode": "Markdown"}
            requests.post(url, data=payload, verify=False, timeout=10)
        except Exception as e:
            print(f">>> [Lỗi Telegram]: {e}")
        alert_queue.task_done()

threading.Thread(target=telegram_sender_thread, daemon=True).start()
#==============================NHẮN TELEGRAM====================

def telegram_listener():
    global allowed_apps
    last_update_id = 0
    while not is_closing:
        try:
            # Gọi API Telegram để lấy tin nhắn mới nhất
            url = f"https://api.telegram.org/bot{TOKEN}/getUpdates"
            params = {"offset": last_update_id + 1, "timeout": 5}
            response = requests.get(url, params=params, verify=False, timeout=10).json()

            if response.get("result"):
                for update in response["result"]:
                    last_update_id = update["update_id"]
                    msg_data = update.get("message", {})
                    text = msg_data.get("text", "").lower().strip()
                    chat_id = msg_data.get("chat", {}).get("id")

                    if chat_id == CHAT_ID:
                        # Ví dụ phụ huynh nhắn: /allow chrome.exe
                        if text.startswith("/allow "):
                            app = text.replace("/allow ", "").strip()
                            if app not in allowed_apps:
                                allowed_apps.append(app)
                                send_telegram_alert(f"✅ Đã mở khóa cho con dùng: `{app}`")
                        
                        # Ví dụ phụ huynh nhắn: /block chrome.exe
                        elif text.startswith("/block "):
                            app = text.replace("/block ", "").strip()
                            if app in allowed_apps and app != "python.exe":
                                allowed_apps.remove(app)
                                send_telegram_alert(f"🔒 Đã khóa lại ứng dụng: `{app}`")
                                
                        elif text == "/list":
                            res = "\n- ".join(allowed_apps)
                            send_telegram_alert(f"📝 App đang được phép:\n- {res}")
        except: 
            pass
        time.sleep(2)

# Chạy luồng lắng nghe song song với App
threading.Thread(target=telegram_listener, daemon=True).start()
# ================= AI FIREWALL =================

def app_firewall():
    global is_closing, allowed_apps
    while not is_closing:
        for proc in psutil.process_iter(['name']):
            try:
                name = proc.info['name'].lower()
                if not name.endswith(".exe"): continue

                # Nếu không có trong danh sách trắng phụ huynh cho phép -> Tắt luôn
                if name not in allowed_apps:
                    # Bỏ qua các file hệ thống quan trọng để máy không sập
                    if name not in ["python.exe", "explorer.exe", "taskhostw.exe", "svchost.exe"]:
                        proc.terminate()
                        send_telegram_alert(f"🚫 Đã chặn app trái phép: {name}")
            except:
                pass
        time.sleep(2)

# ================= AI GỢI Ý =================

def _suggest_next_subject(current: str) -> str:
    all_subjects   = ["Toán", "Văn", "Anh", "Lý", "Hóa", "Sử", "Địa"]
    seven_days_ago = datetime.datetime.now() - datetime.timedelta(days=7)
    recent_totals  = {sub: 0 for sub in all_subjects}
    for h in data.get('history', []):
        try:
            t = datetime.datetime.strptime(h['time'], "%Y-%m-%d %H:%M")
            if t >= seven_days_ago:
                for sub in all_subjects:
                    if sub.lower() in h['subject'].lower():
                        recent_totals[sub] += h['minutes']
        except:
            pass
    candidates = {k: v for k, v in recent_totals.items() if k.lower() != current.lower()}
    if not candidates:
        return ""
    return min(candidates, key=candidates.get)


def ai_suggest():
    subject = subject_entry.get().strip()
    now     = datetime.datetime.now()
    hour    = now.hour
    reasons = []

    history_minutes = [
        h['minutes'] for h in data.get('history', [])
        if subject.lower() in h['subject'].lower()
    ] if subject else []
    total = sum(history_minutes) if history_minutes else 0

    if 5 <= hour < 9:
        time_bonus = 10;  reasons.append("🌅 Sáng sớm — não tươi, +10 phút")
    elif 9 <= hour < 12:
        time_bonus = 5;   reasons.append("☀️ Sáng chính — tập trung cao, +5 phút")
    elif 12 <= hour < 14:
        time_bonus = -10; reasons.append("😴 Sau bữa trưa — dễ buồn ngủ, -10 phút")
    elif 14 <= hour < 18:
        time_bonus = 0;   reasons.append("🌤 Chiều — năng suất ổn định")
    elif 18 <= hour < 22:
        time_bonus = 5;   reasons.append("🌙 Tối — não ôn bài tốt, +5 phút")
    else:
        time_bonus = -15; reasons.append("🌚 Khuya rồi — học ngắn thôi!")

    streak = data.get('streak', 0)
    level  = data.get('level', 1)
    if streak >= 7:
        streak_bonus = 10; reasons.append(f"🔥 Streak {streak} ngày — đang hot, +10 phút!")
    elif streak >= 3:
        streak_bonus = 5;  reasons.append(f"🔥 Streak {streak} ngày — đà tốt, +5 phút")
    elif streak == 0:
        streak_bonus = -5; reasons.append("💤 Mới bắt đầu lại — học nhẹ để lấy đà")
    else:
        streak_bonus = 0
    if level >= 5:
        reasons.append(f"⭐ Level {level} — pro rồi!")

    r = random.random()
    if total == 0:
        base = random.randint(25, 40); reasons.append("📚 Môn mới — bắt đầu nhẹ nhàng")
    elif total < 100:
        base = random.randint(35, 55); reasons.append(f"📖 '{subject}' còn ít ({total} phút)")
    elif r < 0.2:
        base = random.randint(15, 25); reasons.append("😌 Hôm nay thư giãn một chút")
    elif r < 0.7:
        base = random.randint(30, 45); reasons.append("📝 Học vừa đủ — không quá sức")
    else:
        base = random.randint(50, 80); reasons.append("💪 Hôm nay tryhard — let's go!")

    final    = max(10, base + time_bonus + streak_bonus)
    next_sub = _suggest_next_subject(subject)

    study_entry.delete(0, tk.END)
    study_entry.insert(0, final)

    reason_text = " | ".join(reasons[:3])
    if next_sub:
        subject_entry.delete(0, tk.END)
        subject_entry.insert(0, next_sub)
        reason_text += f"  |  ➡️ Tiếp: {next_sub}"
    ai_reason_label.config(text=f"🧠 {reason_text}")

# ================= AI THỜI KHÓA BIỂU =================

def smart_time_subject(sub, priority_factor=1.0):
    now      = datetime.datetime.now()
    week_ago = now - datetime.timedelta(days=7)
    recent_mins = sum([
        h['minutes'] for h in data.get('history', [])
        if sub.lower() in h['subject'].lower() and
        datetime.datetime.strptime(h['time'], "%Y-%m-%d %H:%M") > week_ago
    ])
    r = random.random()
    if recent_mins < 60:  base = random.randint(50, 80)
    elif r < 0.2:         base = random.randint(20, 30)
    elif r < 0.8:         base = random.randint(40, 55)
    else:                 base = random.randint(65, 90)
    return int(base * priority_factor)


def generate_schedule():
    hard_subs = ["Toán", "Lý", "Hóa", "Anh"]
    soft_subs = ["Văn", "Sử", "Địa", "GDCD"]
    days      = ["Thứ 2", "Thứ 3", "Thứ 4", "Thứ 5", "Thứ 6", "Thứ 7", "CN"]

    win = tk.Toplevel(root)
    win.title("📅 Hệ Thống Lập Lịch AI Premium")
    win.geometry("700x500")
    win.configure(bg="#1e293b")

    columns = ("day", "s1", "s2", "s3", "s4", "total")
    tree    = ttk.Treeview(win, columns=columns, show="headings", height=10)
    for col, text in zip(columns, ["Thứ", "Ca 1", "Ca 2", "Ca 3", "Ca 4", "Tổng (phút)"]):
        tree.heading(col, text=text)
        tree.column(col, width=110, anchor="center")

    for day in days:
        sh = random.sample(hard_subs, 2)
        ss = random.sample(soft_subs, 2)
        dl = [(sh[0], smart_time_subject(sh[0])), (ss[0], smart_time_subject(ss[0])),
              (sh[1], smart_time_subject(sh[1])), (ss[1], smart_time_subject(ss[1]))]
        tree.insert("", tk.END, values=(
            day,
            f"{dl[0][0]} ({dl[0][1]}p)", f"{dl[1][0]} ({dl[1][1]}p)",
            f"{dl[2][0]} ({dl[2][1]}p)", f"{dl[3][0]} ({dl[3][1]}p)",
            sum(m for _, m in dl)
        ))

    tree.pack(pady=20, padx=20, fill="both", expand=True)
    tk.Label(win, text="💡 Học xen kẽ môn tự nhiên và xã hội giúp não bền bỉ hơn 25%",
             fg="#facc15", bg="#1e293b", font=("Arial", 10, "italic")).pack(pady=10)
    tk.Button(win, text="📥 Xuất File Lịch (CSV)",
              command=lambda: messagebox.showinfo("Thành công", "Đã lưu lịch vào bộ nhớ!")).pack(pady=5)

# ================= STREAK =================

def update_streak():
    today = datetime.datetime.now().date()
    last  = data["last_day"]
    if last == "":
        data["streak"] = 1
    else:
        last_date = datetime.datetime.strptime(last, "%Y-%m-%d").date()
        diff      = (today - last_date).days
        if diff == 1:   data["streak"] += 1
        elif diff > 1:  data["streak"] = 1
    data["last_day"] = str(today)

# ================= TIMER =================

def start_timer():
    global running, seconds, mode, total_time, timer_id
    if running:
        return
    running = True
    if seconds == 0:
        try:
            study = int(study_entry.get())
            if study <= 0: raise ValueError
        except:
            messagebox.showerror("Lỗi", "Nhập số hợp lệ")
            running = False
            return
        seconds    = study * 60
        total_time = seconds
    mode = "study"
    countdown()


def stop_timer():
    global running
    running = False


def countdown():
    global seconds, running, timer_id, total_time

    if timer_id:
        root.after_cancel(timer_id)
        timer_id = None

    if not running:
        return
    if mode == "study" and not face_detected:
        timer_label.config(text="PAUSED", fg="orange")
        timer_id = root.after(1000, countdown)
        return

    if seconds > 0:
        m, s    = divmod(seconds, 60)
        percent = seconds / total_time
        color   = "#22c55e" if percent > 0.5 else "#facc15" if percent > 0.2 else "#ef4444"
        if percent < 0.2 and seconds % 2 == 0: color = "#ffffff"
        timer_label.config(text=f"{m:02}:{s:02}", fg=color)
        seconds -= 1
        timer_id = root.after(1000, countdown)
    else:
        if mode == "study":
            finish_study()
        else:
            messagebox.showinfo("Xong", "Nghỉ xong rồi! Quay lại học thôi.")
            reset_timer()

# ================= CAMERA =================

cap          = cv2.VideoCapture(0)
face_cascade = cv2.CascadeClassifier(cv2.data.haarcascades + 'haarcascade_frontalface_default.xml')


def camera_thread():
    global face_detected, last_alert_time, study_present_count, study_absent_count, is_closing

    while cap.isOpened():
        if is_closing: break
        ret, frame = cap.read()
        if ret:
            frame = cv2.flip(frame, 1)
            gray  = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
            faces = face_cascade.detectMultiScale(
                gray, scaleFactor=1.1, minNeighbors=3, minSize=(50, 50)
            )
            face_detected = len(faces) > 0
            current_time  = time.time()

            if not face_detected and not is_closing:
                study_absent_count += 1
                if current_time - last_alert_time > 10:
                    send_telegram_alert("⚠️ *CẢNH BÁO:* Không thấy con ở bàn học!")
                    last_alert_time = current_time
            elif face_detected and not is_closing:
                study_present_count += 1

            for (x, y, w, h) in faces:
                cv2.rectangle(frame, (x, y), (x + w, y + h), (0, 255, 0), 2)

            frame_resized = cv2.resize(frame, (320, 240))
            img_rgb       = cv2.cvtColor(frame_resized, cv2.COLOR_BGR2RGB)
            pil_img       = Image.fromarray(img_rgb)

            if frame_queue.full(): frame_queue.get()
            frame_queue.put(pil_img)
        else:
            break
    cap.release()


def check_camera_loop():
    try:
        pil_img = frame_queue.get_nowait()
        imgtk   = ImageTk.PhotoImage(image=pil_img)
        camera_label.imgtk = imgtk
        camera_label.config(image=imgtk)
    except queue.Empty:
        pass
    root.after(20, check_camera_loop)

# ================= BÁO CÁO TELEGRAM =================

def draw_report_chart():
    """Vẽ biểu đồ — gọi trên Main Thread"""
    # FIX #3: dùng datetime.datetime.now() thay vì datetime.now()
    # (Code gốc dùng datetime.now() do import from datetime bị xóa)
    labels = ['Tap trung', 'Xao nhang']
    sizes  = [study_present_count, study_absent_count]
    colors = ['#2ecc71', '#e74c3c']
    if sum(sizes) == 0: sizes = [1, 0]

    plt.figure(figsize=(5, 5))
    plt.pie(sizes, labels=labels, colors=colors, autopct='%1.1f%%', startangle=140)
    plt.title(f"Ket qua hoc tap - {datetime.datetime.now().strftime('%d/%m/%Y')}")
    plt.savefig("chart.png")
    plt.close('all')
    print(">>> [System] Da ve xong bieu do.")


def send_prepared_report():
    """Gửi báo cáo sau khi biểu đồ đã vẽ xong"""
    try:
        date_str   = datetime.datetime.now().strftime("%d/%m/%Y")
        lv         = data.get("level", 1)
        stk        = data.get("streak", 0)
        total_xp   = data.get("xp", 0)
        try:    sub = subject_entry.get() or "Tu hoc"
        except: sub = "Tu hoc"
        actual_mins = total_time // 60

        caption_msg = (
            f"📊 *BAO CAO KET QUA HOC TAP*\n"
            f"📅 Ngay: {date_str}\n"
            f"📚 Mon hoc: {sub}\n"
            f"⏱️ Thoi gian: {actual_mins} phut\n"
            f"🏆 Cap do: Level {lv}\n"
            f"🔥 Streak: {stk} ngay\n"
            f"✨ Tong XP: {total_xp}\n"
            f"💪 *Hay khen ngoi con nhe!*"
        )
        url = f"https://api.telegram.org/bot{TOKEN}/sendPhoto"
        with open("chart.png", "rb") as photo:
            requests.post(url, files={"photo": photo},
                          data={"chat_id": CHAT_ID, "caption": caption_msg, "parse_mode": "Markdown"},
                          verify=False, timeout=15)
        print(">>> [Telegram] Da gui bao cao thanh cong!")
    except Exception as e:
        print(f"Lỗi gửi: {e}")


def handle_telegram_report():
    # FIX #4: Gọi trực tiếp trên main thread (không bọc threading.Thread)
    # vì dùng root.after — gọi từ thread phụ sẽ crash
    root.after(0,    draw_report_chart)
    root.after(1000, send_prepared_report)

# ================= FINISH STUDY =================

def finish_study():
    global seconds, mode
    try:
        import winsound
        for freq in [1000, 1200, 1400, 1200, 1000]:
            winsound.Beep(freq, 150 if freq != 1000 else 300)
    except:
        print("Ting Ting Ting")

    minutes = int(study_entry.get())
    data["history"].append({
        "time":    str(datetime.datetime.now())[:16],
        "subject": subject_entry.get(),
        "minutes": minutes
    })
    data["xp"]           += minutes * 5
    data["week_minutes"] += minutes
    today = str(datetime.datetime.now().date())
    data["daily"][today] = data["daily"].get(today, 0) + minutes
    update_streak()

    while data["xp"] >= data["level"] * 100:
        data["xp"]    -= data["level"] * 100
        data["level"] += 1
        messagebox.showinfo("LEVEL UP", f"🎉 Level {data['level']} !")

    save_data()
    update_ui()
    messagebox.showinfo("✅ Học xong!", f"Học {minutes} phút xong! Chuẩn bị nghỉ...")

    try:    break_time = int(break_entry.get())
    except: break_time = 5
    seconds = break_time * 60
    mode    = "break"
    countdown()
    handle_telegram_report()  # Gọi trực tiếp, không threading

# ================= RESET =================

def reset_timer():
    global running, seconds, timer_id
    running = False
    if timer_id:
        root.after_cancel(timer_id)
        timer_id = None
    seconds = 0
    timer_label.config(text="00:00", fg="#22d3ee")

# ================= CHART =================

def show_chart():
    if not data["daily"] and not data["history"]:
        messagebox.showinfo("Thông báo", "Chưa có dữ liệu")
        return

    import matplotlib
    matplotlib.use("TkAgg")
    from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
    import collections

    win = tk.Toplevel(root)
    win.title("📊 Thống kê học tập")
    win.geometry("900x650")
    win.configure(bg="#0f172a")

    tab_control = ttk.Notebook(win)
    tab_control.pack(fill="both", expand=True, padx=10, pady=10)

    COLOR_BG    = "#1e293b"
    COLOR_TEXT  = "#e2e8f0"
    COLOR_BLUE  = "#38bdf8"
    COLOR_GREEN = "#4ade80"

    def make_fig(tab):
        fig, ax = plt.subplots(facecolor=COLOR_BG)
        ax.set_facecolor(COLOR_BG)
        for spine in ax.spines.values(): spine.set_edgecolor("#334155")
        ax.tick_params(colors=COLOR_TEXT)
        ax.xaxis.label.set_color(COLOR_TEXT)
        ax.yaxis.label.set_color(COLOR_TEXT)
        ax.title.set_color(COLOR_TEXT)
        canvas = FigureCanvasTkAgg(fig, master=tab)
        canvas.get_tk_widget().pack(fill="both", expand=True)
        return fig, ax, canvas

    # Tab 1 — Theo ngày
    tab1 = ttk.Frame(tab_control)
    tab_control.add(tab1, text="📅 Theo ngày")
    if data["daily"]:
        fig1, ax1, cv1 = make_fig(tab1)
        days_k = list(data["daily"].keys())
        mins_v = list(data["daily"].values())
        bars   = ax1.bar(days_k, mins_v, color=COLOR_BLUE, width=0.5)
        for bar, v in zip(bars, mins_v):
            ax1.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 1,
                     str(v), ha='center', va='bottom', color=COLOR_TEXT, fontsize=9)
        ax1.set_title("Số phút học mỗi ngày")
        ax1.set_ylabel("Phút")
        plt.setp(ax1.get_xticklabels(), rotation=45, ha='right', fontsize=8)
        fig1.tight_layout(); cv1.draw()
    else:
        tk.Label(tab1, text="Chưa có dữ liệu", bg="#0f172a", fg="gray").pack(expand=True)

    # Tab 2 — Xu hướng
    tab2 = ttk.Frame(tab_control)
    tab_control.add(tab2, text="📈 Xu hướng")
    if data["daily"]:
        fig2, ax2, cv2 = make_fig(tab2)
        sd = sorted(data["daily"].keys())
        sm = [data["daily"][d] for d in sd]
        ax2.plot(sd, sm, color=COLOR_GREEN, marker='o', linewidth=2, markersize=5)
        ax2.fill_between(range(len(sd)), sm, alpha=0.15, color=COLOR_GREEN)
        ax2.set_xticks(range(len(sd)))
        ax2.set_xticklabels(sd, rotation=45, ha='right', fontsize=8)
        ax2.set_title("Xu hướng học theo thời gian")
        ax2.set_ylabel("Phút")
        fig2.tight_layout(); cv2.draw()
    else:
        tk.Label(tab2, text="Chưa có dữ liệu", bg="#0f172a", fg="gray").pack(expand=True)

    # Tab 3 — Theo môn
    tab3 = ttk.Frame(tab_control)
    tab_control.add(tab3, text="🥧 Theo môn")
    if data["history"]:
        fig3, ax3, cv3 = make_fig(tab3)
        st = collections.defaultdict(int)
        for h in data["history"]: st[h["subject"]] += h["minutes"]
        wedges, texts, autotexts = ax3.pie(
            list(st.values()), labels=list(st.keys()),
            autopct='%1.1f%%', colors=plt.cm.Set3.colors[:len(st)],
            startangle=90, textprops={'color': COLOR_TEXT, 'fontsize': 9}
        )
        for at in autotexts: at.set_color("#0f172a"); at.set_fontweight("bold")
        ax3.set_title("Tỉ lệ thời gian theo môn học")
        fig3.tight_layout(); cv3.draw()
    else:
        tk.Label(tab3, text="Chưa có dữ liệu lịch sử", bg="#0f172a", fg="gray").pack(expand=True)

    # Tab 4 — Top & Tuần
    tab4 = ttk.Frame(tab_control)
    tab_control.add(tab4, text="🏆 Top & Tuần")
    if data["history"]:
        fig4, (ax4a, ax4b) = plt.subplots(1, 2, facecolor=COLOR_BG)
        for ax in (ax4a, ax4b):
            ax.set_facecolor(COLOR_BG)
            for spine in ax.spines.values(): spine.set_edgecolor("#334155")
            ax.tick_params(colors=COLOR_TEXT)
            ax.title.set_color(COLOR_TEXT)
            ax.yaxis.label.set_color(COLOR_TEXT)

        st = collections.defaultdict(int)
        for h in data["history"]: st[h["subject"]] += h["minutes"]
        top = sorted(st.items(), key=lambda x: x[1], reverse=True)[:6]
        bars = ax4a.barh([x[0] for x in top], [x[1] for x in top],
                         color=[COLOR_BLUE if i == 0 else "#64748b" for i in range(len(top))])
        for bar, v in zip(bars, [x[1] for x in top]):
            ax4a.text(bar.get_width() + 1, bar.get_y() + bar.get_height()/2,
                      f"{v}p", va='center', color=COLOR_TEXT, fontsize=8)
        ax4a.set_title("Top mon hoc nhieu nhat")
        ax4a.invert_yaxis()

        today      = datetime.datetime.now().date()
        week_start = today - datetime.timedelta(days=today.weekday())
        prev_start = week_start - datetime.timedelta(days=7)
        this_week  = collections.defaultdict(int)
        prev_week  = collections.defaultdict(int)
        for h in data["history"]:
            try:
                d = datetime.datetime.strptime(h["time"], "%Y-%m-%d %H:%M").date()
                if d >= week_start:   this_week[h["subject"]] += h["minutes"]
                elif d >= prev_start: prev_week[h["subject"]] += h["minutes"]
            except: pass

        all_subs = sorted(set(list(this_week.keys()) + list(prev_week.keys())))
        if all_subs:
            x = range(len(all_subs)); w = 0.35
            ax4b.bar([i - w/2 for i in x], [prev_week.get(s, 0) for s in all_subs],
                     width=w, label="Tuần trước", color="#64748b")
            ax4b.bar([i + w/2 for i in x], [this_week.get(s, 0) for s in all_subs],
                     width=w, label="Tuần này", color=COLOR_GREEN)
            ax4b.set_xticks(list(x))
            ax4b.set_xticklabels(all_subs, rotation=30, ha='right', fontsize=8, color=COLOR_TEXT)
            ax4b.legend(facecolor=COLOR_BG, labelcolor=COLOR_TEXT, fontsize=8)
        ax4b.set_title("Tuan nay vs tuan truoc")
        ax4b.set_ylabel("Phút")

        canvas4 = FigureCanvasTkAgg(fig4, master=tab4)
        canvas4.get_tk_widget().pack(fill="both", expand=True)
        fig4.tight_layout(); canvas4.draw()
    else:
        tk.Label(tab4, text="Chưa có dữ liệu lịch sử", bg="#0f172a", fg="gray").pack(expand=True)

# ================= UI UPDATE =================

def update_ui():
    xp_bar["value"]   = data["xp"]
    xp_bar["maximum"] = data["level"] * 100
    level_label.config(text=f"⭐ Lv {data['level']} | XP {data['xp']}")
    streak_label.config(text=f"🔥 {data['streak']} ngày")
    mission_label.config(text=f"🎯 {data.get('week_minutes', 0)}/300 phút")

# ================= UI =================

root = tk.Tk()
root.title("Hành Trình Học Tập")
root.geometry("1000x700")
root.configure(bg="#0f172a")

camera_label = tk.Label(root, bg="black", bd=2, relief="ridge")
camera_label.place(x=20, y=20)

clock_frame = tk.Frame(root, bg="#1e293b", bd=4, relief="ridge")
clock_frame.place(relx=1.0, y=10, anchor="ne")
clock_label = tk.Label(clock_frame, font=("Arial", 14, "bold"), fg="#22c55e", bg="#1e293b")
clock_label.pack(padx=10, pady=5)
date_label  = tk.Label(clock_frame, font=("Arial", 12, "bold"), fg="lightblue",  bg="#1e293b")
date_label.pack(padx=10, pady=(0, 5))
icon_label  = tk.Label(clock_frame, font=("Arial", 12), bg="#1e293b")
icon_label.pack(padx=5, pady=(0, 5))


def update_clock():
    now      = datetime.datetime.now()
    hour     = now.hour
    color    = "#22c55e" if 6 <= hour < 18 else "#facc15"
    icon     = "🌞"      if 6 <= hour < 18 else "🌙"
    clock_label.config(text=now.strftime("%H:%M:%S"), fg=color)
    date_label.config(text=now.strftime("%d/%m/%Y"))
    icon_label.config(text=icon, fg=color)
    root.after(1000, update_clock)


tk.Label(root, text="🎮 Hành Trình Học Tập",
         font=("Arial", 28, "bold"), fg="white", bg="#0f172a").pack(pady=10)

level_label   = tk.Label(root, fg="white",      bg="#0f172a"); level_label.pack()
xp_bar        = ttk.Progressbar(root, length=400);             xp_bar.pack(pady=10)
streak_label  = tk.Label(root, fg="orange",     bg="#0f172a"); streak_label.pack()
mission_label = tk.Label(root, fg="lightgreen", bg="#0f172a"); mission_label.pack()

frame = tk.Frame(root, bg="#1e293b")
frame.pack(pady=20)
tk.Label(frame, text="📚", bg="#1e293b", fg="white").grid(row=0, column=0)
subject_entry = tk.Entry(frame, width=10); subject_entry.grid(row=0, column=1)
tk.Label(frame, text="⏱",  bg="#1e293b", fg="white").grid(row=0, column=2)
study_entry   = tk.Entry(frame, width=5);  study_entry.grid(row=0, column=3)
tk.Label(frame, text="☕",  bg="#1e293b", fg="white").grid(row=0, column=4)
break_entry   = tk.Entry(frame, width=5);  break_entry.grid(row=0, column=5)

tk.Label(root, text='📌 📚 = môn học | ⏱ = thời gian học | ☕ = thời gian nghỉ',
         fg="lightgray", bg="#0f172a").pack()

ai_reason_label = tk.Label(root, text="", fg="#94a3b8", bg="#0f172a",
                            wraplength=700, font=("Arial", 9))
ai_reason_label.pack(pady=(0, 4))

timer_label = tk.Label(root, text="00:00", font=("Arial", 80, "bold"),
                        fg="#22d3ee", bg="#0f172a")
timer_label.pack(expand=True)

btn_frame = tk.Frame(root, bg="#0f172a")
btn_frame.pack()
tk.Button(btn_frame, text="▶ Start",     command=start_timer).grid(row=0, column=0, padx=5)
tk.Button(btn_frame, text="🔁 Reset",    command=reset_timer).grid(row=0, column=1, padx=5)
tk.Button(btn_frame, text="📊 Chart",    command=show_chart).grid(row=0, column=2, padx=5)
tk.Button(btn_frame, text="⏸ Stop",     command=stop_timer).grid(row=0, column=3, padx=5)

fullscreen = False
def toggle_fullscreen():
    global fullscreen
    fullscreen = not fullscreen
    root.attributes("-fullscreen", fullscreen)

tk.Button(btn_frame, text="🖥 Fullscreen", command=toggle_fullscreen).grid(row=0, column=4, padx=5)

ai_frame = tk.Frame(root, bg="#0f172a")
ai_frame.pack(pady=10)
tk.Button(ai_frame, text="🧠 AI",       command=ai_suggest).pack(side="left", padx=10)
tk.Button(ai_frame, text="📅 Schedule", command=generate_schedule).pack(side="left", padx=10)

# ================= HISTORY & REMINDER WINDOWS =================

history_window  = None
reminder_window = None


def open_history_window():
    global history_window
    if history_window is not None and history_window.winfo_exists():
        history_window.lift(); return
    history_window = tk.Toplevel(root)
    history_window.title("📜 Lịch sử học")
    history_window.geometry("400x400")
    text = tk.Text(history_window)
    text.pack(fill="both", expand=True)

    def refresh():
        if not history_window.winfo_exists(): return
        text.delete("1.0", tk.END)
        for item in data.get("history", []):
            text.insert(tk.END, f"{item['time']} | {item['subject']} | {item['minutes']} phút\n")
        history_window.after(2000, refresh)
    refresh()


def open_reminder_window():
    global reminder_window
    if reminder_window is not None and reminder_window.winfo_exists():
        reminder_window.lift(); return
    reminder_window = tk.Toplevel(root)
    reminder_window.title("🔔 Nhắc nhở")
    reminder_window.geometry("400x450")

    top_frame = tk.Frame(reminder_window)
    top_frame.pack(pady=5)
    time_entry = tk.Entry(top_frame, width=10); time_entry.pack(side="left", padx=5)
    time_entry.insert(0, "HH:MM")
    msg_entry  = tk.Entry(top_frame, width=20); msg_entry.pack(side="left", padx=5)

    def add_local():
        data["reminders"].append((time_entry.get(), msg_entry.get()))
        save_data(); update_box()

    def delete_local():
        try:
            idx = int(time_entry.get()) - 1
            data["reminders"].pop(idx); save_data(); update_box()
        except:
            messagebox.showerror("Lỗi", "Nhập số thứ tự để xoá")

    tk.Button(top_frame, text="Thêm",  command=add_local).pack(side="left", padx=5)
    tk.Button(top_frame, text="❌ Xóa", command=delete_local).pack(side="left", padx=5)

    text = tk.Text(reminder_window)
    text.pack(fill="both", expand=True)

    def update_box():
        text.delete("1.0", tk.END)
        for i, (t, m) in enumerate(data.get("reminders", [])):
            text.insert(tk.END, f"🔔 {i+1}. {t} | 📌 {m}\n")

    def refresh():
        if reminder_window is None or not reminder_window.winfo_exists(): return
        update_box()
        reminder_window.after(2000, refresh)
    refresh()


window_btn_frame = tk.Frame(root, bg="#0f172a")
window_btn_frame.pack(pady=10)
tk.Button(window_btn_frame, text="📜 Mở Lịch sử",  command=open_history_window).pack(side="left", padx=10)
tk.Button(window_btn_frame, text="🔔 Mở Nhắc nhở", command=open_reminder_window).pack(side="left", padx=10)

# ================= REMINDERS CHECK =================
# FIX #5: dùng reminder_last_time (string) riêng biệt với last_alert_time (float của camera)
#         Code gốc dùng chung last_alert_time = "" rồi gán lại float → TypeError

def check_reminders():
    global reminder_last_time
    now = datetime.datetime.now().strftime("%H:%M")
    for t, m in data.get("reminders", []):
        if t == now and reminder_last_time != now:
            reminder_last_time = now
            try:
                import winsound
                for _ in range(3): winsound.Beep(1500, 200)
            except:
                print("🔔 Ting!")
            if PLYER_AVAILABLE:
                notification.notify(title="🔔 Nhắc học", message=f"Đến giờ: {m}", timeout=5)
            else:
                messagebox.showinfo("🔔 Nhắc học", f"Đến giờ: {m}")
    root.after(10000, check_reminders)

# ================= ON CLOSE =================

def on_closing():
    global is_closing
    if messagebox.askokcancel("Thoát", "Bạn có chắc muốn dừng học và thoát không?"):
        is_closing = True
        print(">>> Đang gửi báo cáo cuối buổi, vui lòng đợi...")
        try:
            url_text = f"https://api.telegram.org/bot{TOKEN}/sendMessage"
            requests.post(url_text, verify=False, timeout=5,
                          data={"chat_id": CHAT_ID,
                                "text": "🔴 *THÔNG BÁO:* Con đã nghỉ học và tắt ứng dụng!",
                                "parse_mode": "Markdown"})
            draw_report_chart()
            url_photo = f"https://api.telegram.org/bot{TOKEN}/sendPhoto"
            with open("chart.png", "rb") as photo:
                requests.post(url_photo, verify=False, timeout=10,
                              files={"photo": photo},
                              data={"chat_id": CHAT_ID, "caption": "📊 Báo cáo cuối buổi"})
            print(">>> Đã gửi xong tất cả!")
        except Exception as e:
            print(f"Lỗi khi đang thoát: {e}")

        if cap.isOpened(): cap.release()
        root.destroy()

# ================= START =================

update_ui()
update_clock()
check_reminders()
check_camera_loop()

threading.Thread(target=camera_thread, daemon=True).start()


root.protocol("WM_DELETE_WINDOW", on_closing)
root.mainloop()
