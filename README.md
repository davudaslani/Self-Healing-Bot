# Self-Healing-Bot

## 1. Projektübersicht

Der **Self-Healing Bot** ist ein automatisierter Administrations-Bot, der über **Telegram** gesteuert wird.  
Er überwacht den Zustand eines Linux-Systems und kann bei Problemen selbstständig Massnahmen ergreifen.

Das Projekt verbindet mehrere Informatik-Bereiche:
- Systemadministration
- Automatisierung
- IT-Sicherheit
- Programmierung mit Python
- Client–Server-Kommunikation (Telegram API)

---

## 2. Ziel des Projekts

Ziel war es, ein System zu entwickeln, das:
- den **Systemzustand überwacht**
- **Fehler automatisch erkennt**
- **Self-Healing-Massnahmen** ausführt
- über ein **einfaches Interface (Telegram)** bedient werden kann

Solche Systeme werden in der Praxis z. B. in Rechenzentren, Cloud-Umgebungen oder Serverüberwachung eingesetzt.

---

## 3. Verwendete Technologien

| Technologie | Zweck |
|------------|------|
| Python | Hauptprogrammiersprache |
| Telegram Bot API | Benutzeroberfläche |
| python-telegram-bot | Kommunikation mit Telegram |
| Linux Systemdienste | Überwachung & Reparatur |
| dotenv | Sichere Verwaltung von Zugangsdaten |
| Subprocess | Ausführen von Systembefehlen |

---

## 4. Code-Erklärung

Hier erkläre ich die Code-Dateien die ich programmiert habe, damit dieses Projekt korrekt funktioniert.

**bot.py**

```python
import os  # Zugriff auf Umgebungsvariablen und Systemfunktionen
import subprocess  # Zum Ausführen von Systembefehlen (z.B. reboot)
from dotenv import load_dotenv  # Zum Laden der .env Datei (Token, User-ID)
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup  # Telegram Bot API Objekte
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes, CallbackQueryHandler  # Telegram Bot Handler

# Eigene Module importieren
from modules.checks.status import get_system_status  # Funktion zum Abfragen von CPU, RAM, Disk, Services
from modules.checks.health_check import run_all_checks  # Führt alle Self-Healing Checks aus
from modules.checks.update_manager import check_updates, run_updates  # Update-Abfragen und Installation
from config import LOG_FILE  # Pfad zur Log-Datei

# ----------------------------
# Environment Variables laden
# ----------------------------
load_dotenv()  # .env Datei laden
BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")  # Bot Token aus .env
AUTHORIZED_USER_ID = int(os.getenv("AUTHORIZED_USER_ID"))  # Erlaubte Telegram User-ID

# ----------------------------
# Zugriffsschutz Funktion
# ----------------------------
def is_authorized(user_id: int) -> bool:
    # Prüft, ob der Benutzer autorisiert ist
    return user_id == AUTHORIZED_USER_ID

# ----------------------------
# Bot Commands
# ----------------------------

# /start Befehl
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):  # Zugriff prüfen
        await update.message.reply_text("❌ Zugriff verweigert.")  # Nachricht, wenn nicht autorisiert
        return
    await update.message.reply_text("✅ Bot läuft! Du bist autorisiert.")  # Erfolgsmeldung

# /help Befehl
async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("❌ Zugriff verweigert.")
        return

    # Übersicht aller Befehle
    text = (
        "🧠 *Self-Healing Bot – Befehle*\n\n"
        "/start – Check ob Bot läuft\n"
        "/help – Befehlsübersicht\n"
        "/status – Zeigt Systemstatus (CPU, RAM, Disk, Services)\n"
        "/heal – Führt Self-Healing Checks aus\n"
        "/logs – Sendet Log-Datei\n"
        "/updates – Zeigt verfügbare Updates\n"
        "/update_now – Führt Updates aus\n"
        "/reboot – System neustarten\n"
        "/menu – Dashboard mit Buttons"
    )
    await update.message.reply_text(text, parse_mode="Markdown")  # Nachricht mit Markdown formatieren

# /status Befehl
async def status_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("❌ Zugriff verweigert.")
        return
    
    # Systemstatus abrufen
    cpu, ram, disk, services = get_system_status()
    msg = (
        "📊 *Systemstatus*\n\n"
        f"🖥 CPU: {cpu}%\n"
        f"📦 RAM: {ram}%\n"
        f"💽 Disk: {disk:.2f}%\n\n"
        f"📍 *Services:*\n{services}"
    )
    await update.message.reply_text(msg, parse_mode="Markdown")  # Nachricht senden

# /heal Befehl
async def heal_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("❌ Zugriff verweigert.")
        return

    run_all_checks()  # Alle Self-Healing Checks ausführen
    await update.message.reply_text("🧠 Healing ausgeführt! ✅\n(Logs wurden aktualisiert)")

# /logs Befehl
async def logs_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("❌ Zugriff verweigert.")
        return
    
    # Log-Datei als Dokument senden
    with open(LOG_FILE, "rb") as f:
        await update.message.reply_document(f)

# ----------------------------
# Hilfsfunktion für lange Texte (Updates)
# ----------------------------
def split_text(text, limit=4000):
    """
    Telegram Nachrichten haben Limit (~4096 Zeichen)
    -> lange Texte in mehrere Nachrichten splitten
    """
    lines = text.splitlines()
    chunks = []
    current = ""
    for line in lines:
        if len(current) + len(line) + 1 > limit:
            chunks.append(current)
            current = ""
        current += line + "\n"
    if current:
        chunks.append(current)
    return chunks

# /updates Befehl
async def updates_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("❌ Zugriff verweigert.")
        return
    
    # Updates prüfen
    apt_updates, snap_updates = check_updates()
    apt_updates = apt_updates or "Keine Updates verfügbar"
    snap_updates = snap_updates or "Keine Updates verfügbar"
    
    # Text splitten, wenn zu lang
    for chunk in split_text("*APT Updates:*\n" + apt_updates, 4000):
        await update.message.reply_text(chunk, parse_mode="Markdown")
    for chunk in split_text("*Snap Updates:*\n" + snap_updates, 4000):
        await update.message.reply_text(chunk, parse_mode="Markdown")

# /update_now Befehl
async def update_now_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("❌ Zugriff verweigert.")
        return
    
    await update.message.reply_text("🔄 Updates werden installiert...")
    try:
        result = run_updates()  # Updates installieren
        await update.message.reply_text(result)
    except Exception as e:
        await update.message.reply_text(f"❌ Fehler beim Update:\n{e}")

# /reboot Befehl
async def reboot_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("❌ Zugriff verweigert.")
        return
    
    await update.message.reply_text("🔁 System wird neu gestartet...")
    subprocess.run(["sudo", "reboot"])  # System neu starten

# ----------------------------
# Dashboard / Inline Buttons
# ----------------------------
async def menu_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update.effective_user.id):
        await update.message.reply_text("❌ Zugriff verweigert.")
        return

    # Buttons erstellen
    keyboard = [
        [InlineKeyboardButton("📊 Status", callback_data="status")],
        [InlineKeyboardButton("🧠 Heal", callback_data="heal")],
        [InlineKeyboardButton("📄 Logs", callback_data="logs")],
        [InlineKeyboardButton("🔄 Updates prüfen", callback_data="updates")],
        [InlineKeyboardButton("⬆️ Updates installieren", callback_data="update_now")],
        [InlineKeyboardButton("🔁 Reboot", callback_data="reboot")]
    ]

    reply_markup = InlineKeyboardMarkup(keyboard)  # Keyboard ins Telegram Format bringen
    await update.message.reply_text("🖥 Self-Healing Bot Dashboard:", reply_markup=reply_markup)

# Callback für Buttons
async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()  # Klick bestätigen

    # Button-Aktionen auf Commands mappen
    if query.data == "status":
        await status_cmd(update, context)
    elif query.data == "heal":
        await heal_cmd(update, context)
    elif query.data == "logs":
        await logs_cmd(update, context)
    elif query.data == "updates":
        await updates_cmd(update, context)
    elif query.data == "update_now":
        await update_now_cmd(update, context)
    elif query.data == "reboot":
        await reboot_cmd(update, context)

# ----------------------------
# Main
# ----------------------------
if __name__ == "__main__":
    app = ApplicationBuilder().token(BOT_TOKEN).build()  # Bot erstellen
    
    # Command Handler registrieren
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_cmd))
    app.add_handler(CommandHandler("status", status_cmd))
    app.add_handler(CommandHandler("heal", heal_cmd))
    app.add_handler(CommandHandler("logs", logs_cmd))
    app.add_handler(CommandHandler("updates", updates_cmd))
    app.add_handler(CommandHandler("update_now", update_now_cmd))
    app.add_handler(CommandHandler("reboot", reboot_cmd))
    app.add_handler(CommandHandler("menu", menu_cmd))
    
    # Callback Handler für Dashboard Buttons
    app.add_handler(CallbackQueryHandler(button_callback))
    
    print("🤖 Bot läuft... (mit Long Polling und Dashboard Buttons)")
    app.run_polling()  # Bot startet und hört auf Telegram Nachrichten

```

Der Code startet einen Telegram-Bot, der nur für einen autorisierten Benutzer zugänglich ist. Er kann über verschiedene Befehle oder ein Dashboard:

- den Systemstatus anzeigen (CPU, RAM, Festplatte, Services),
- Self-Healing Checks ausführen, um Probleme automatisch zu beheben,
- die Log-Datei senden,
- System-Updates prüfen und installieren,
- das System neu starten.

**config.py**

```python
# ----------------------------
# Liste der Systemdienste, die vom Bot überwacht werden
# ----------------------------
SERVICES_TO_MONITOR = [
    "NetworkManager",      # Netzwerkverwaltung
    "ssh",                 # SSH-Dienst (Fernzugriff)
    "cron",                # Zeitgesteuerte Aufgaben
    "bluetooth",           # Bluetooth-Dienst
    "cups",                # Drucksystem
    "systemd-timesyncd",   # Zeit-Synchronisation
    "ufw",                 # Firewall
    "snapd",               # Snap-Paketverwaltung
    "avahi-daemon",        # Netzwerk-Dienst für lokale Namensauflösung
    "docker",              # Docker-Dienst (Containerverwaltung)
]

# ----------------------------
# Festlegen der Warnschwelle für die Festplattennutzung
# ----------------------------
DISK_USAGE_THRESHOLD = 85  # Prozent – ab diesem Wert kann der Bot Warnungen ausgeben

# ----------------------------
# Pfad zur Log-Datei, in der alle Aktionen protokolliert werden
# ----------------------------
LOG_FILE = "logs/actions.log"
```

Diese Konfiguration legt fest:

- Welche Dienste auf dem System überwacht werden `(SERVICES_TO_MONITOR)`.
- Bei welcher Festplattenauslastung der Bot Alarm schlagen soll `(DISK_USAGE_THRESHOLD)`.
- Wo die Log-Datei gespeichert wird, in der alle Aktionen und Self-Healing-Massnahmen protokolliert werden `(LOG_FILE)`.

**run_checks.py**

```python
# Importiere die Funktion, die alle Self-Healing Checks ausführt
from modules.checks.health_check import run_all_checks

# ----------------------------
# Main-Bereich
# ----------------------------
if __name__ == "__main__":  # Prüft, ob die Datei direkt ausgeführt wird
    run_all_checks()         # Führt alle definierten Health-Checks aus (Systemdienste, Disk, etc.)
    print("✔ Health-Checks abgeschlossen. Logs unter logs/actions.log")  # Ausgabe, dass Checks fertig sind
```

Dieses Skript führt alle Self-Healing-Checks des Systems aus, wenn es direkt gestartet wird.
Dabei werden Dienste überprüft, Fehler automatisch behoben (falls definiert) und alle Aktionen in der Log-Datei `logs/actions.log` protokolliert. Nach Abschluss gibt das Skript eine Bestätigung in der Konsole aus.

---

## 5. Ordner `/logs/actions.log`

- Zweck: Protokolliert alle Aktionen, die der Self-Healing-Bot durchführt.
- Dazu gehören z. B.:
  - Ergebnisse der Self-Healing-Checks
  - Neustarts von Diensten
  - Installierte Updates
  - Fehler oder Warnungen

- Warum wichtig:
  - Ermöglicht es dem Benutzer, nachzuvollziehen, welche Massnahmen der Bot ergriffen hat.
  - Hilft bei Fehleranalyse und Debugging.
  - Macht das Projekt professionell dokumentiert.

---

> [!IMPORTANT]
>
> Ebenfalls läuft dieses Skript regelmässig, d. h. es wird nie unterbrochen, solange mein WSL aktiv ist.
