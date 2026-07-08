# Day-02-Expense-Tracker
"""
💸 CHAOS WALLET — A Gamified Expense Tracker 💸
------------------------------------------------
Features:
1. Add expenses with a MOOD tag (how you felt when you spent)
2. Financial Damage Meter (your wallet's HP bar for the week)
3. Weekly Spending Streak (days in a row you stayed on budget)
4. Big Money Villain (the spending category that's wrecking you)
5. AI-style Spending Roast (a snarky auto-generated comment)

Everything is saved to a local file called `wallet_data.json`,
so your data survives even after you close the program.
"""

import json          # lets us save/load data as a file (like a mini database)
import os            # lets us check if a file exists
import random         # lets us randomly pick a roast line
from datetime import date, timedelta   # lets us work with today's date, and date math

DATA_FILE = "wallet_data.json"

# Emoji options for moods — a simple dictionary (key -> emoji)
MOODS = {
    "1": ("happy", "😀"),
    "2": ("stressed", "😡"),
    "3": ("sad", "😢"),
    "4": ("bored", "😴"),
    "5": ("anxious", "😬"),
    "6": ("neutral", "🤷"),
}

# Flavor names for the "Big Money Villain" based on category keywords
VILLAIN_NAMES = {
    "food": "The Midnight Snack Overlord 🍔",
    "coffee": "Sir Latte, Duke of Debt ☕",
    "shopping": "The Cart Abandonment... NOT Baron 🛍️",
    "subscriptions": "The Auto-Renewal Phantom 👻",
    "delivery": "The UberEats Emperor 🛵",
    "entertainment": "The Netflix-and-Bill Dragon 🐉",
    "travel": "The Wanderlust Bandit ✈️",
    "rent": "The Landlord Titan 🏰",
}

# Roast lines, grouped by situation. {villain} and {amount} get filled in later.
ROASTS_OVER_BUDGET = [
    "💀 {villain} just walked off with ₹{amount}. Your wallet didn't even see it coming.",
    "🔥 You blew past your budget like it was a suggestion, not a rule.",
    "📉 At this rate, {villain} should just start paying YOUR rent.",
    "🚨 Emergency! Your bank account just texted 'please stop.'",
]
ROASTS_UNDER_BUDGET = [
    "✨ Look at you, actually being responsible. Who ARE you?",
    "🛡️ You fought off {villain} this week. A rare victory!",
    "🌱 Your wallet is healthy, hydrated, and thriving.",
]
ROASTS_MOOD_STRESS = [
    "😬 Noticing a lot of 'stressed' spending... your wallet is basically your emotional support animal.",
    "🎢 Stress-spending detected. Maybe try screaming into a pillow instead of your card next time.",
]


def load_data():
    """Load saved wallet data from the JSON file, or create a fresh one."""
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r") as f:
            return json.load(f)
    # Default structure if no file exists yet
    return {"expenses": [], "weekly_budget": 1000.0}


def save_data(data):
    """Write the current data back to the JSON file so nothing is lost."""
    with open(DATA_FILE, "w") as f:
        json.dump(data, f, indent=2)


def add_expense(data):
    """Ask the user for expense details and mood, then store it."""
    print("\n--- ➕ Add a new expense ---")
    category = input("Category (e.g. food, coffee, shopping): ").strip().lower()

    # Keep asking until we get a valid number
    while True:
        try:
            amount = float(input("Amount spent (₹): "))
            break
        except ValueError:
            print("That's not a number, try again 🙃")

    print("How were you feeling when you spent this?")
    for key, (name, emoji) in MOODS.items():
        print(f"  {key}. {emoji} {name}")
    mood_choice = input("Pick a number: ").strip()
    mood_name, mood_emoji = MOODS.get(mood_choice, ("neutral", "🤷"))

    expense = {
        "date": str(date.today()),   # store date as text like "2026-07-09"
        "category": category,
        "amount": amount,
        "mood": mood_name,
    }
    data["expenses"].append(expense)
    save_data(data)
    print(f"✅ Logged ₹{amount} on '{category}' while feeling {mood_emoji} {mood_name}.")


def expenses_this_week(data):
    """Return only the expenses from the last 7 days."""
    today = date.today()
    week_ago = today - timedelta(days=7)
    result = []
    for e in data["expenses"]:
        e_date = date.fromisoformat(e["date"])
        if week_ago <= e_date <= today:
            result.append(e)
    return result


def show_damage_meter(data):
    """Show a health-bar-style meter representing wallet damage this week."""
    print("\n--- 🩸 Financial Damage Meter ---")
    week_expenses = expenses_this_week(data)
    total_spent = sum(e["amount"] for e in week_expenses)
    budget = data["weekly_budget"]

    # Percentage of budget used, capped at 100% for the bar display
    percent = min(total_spent / budget, 1.0) if budget > 0 else 0
    total_blocks = 10
    filled = round(percent * total_blocks)
    empty = total_blocks - filled

    bar = "🟥" * filled + "🟩" * empty
    print(f"Wallet HP: {bar}")
    print(f"Spent this week: ₹{total_spent:.2f} / ₹{budget:.2f} budget")

    if percent >= 1.0:
        print("💀 Your wallet has taken FATAL damage this week.")
    elif percent >= 0.7:
        print("⚠️ Your wallet is badly wounded. Proceed with caution.")
    else:
        print("🙂 Your wallet is holding up fine.")

    return total_spent, budget


def calculate_streak(data):
    """Count consecutive days (ending today) where daily spend stayed under the daily budget."""
    daily_budget = data["weekly_budget"] / 7
    streak = 0
    day = date.today()

    while True:
        day_total = sum(
            e["amount"] for e in data["expenses"] if e["date"] == str(day)
        )
        if day_total <= daily_budget:
            streak += 1
            day = day - timedelta(days=1)   # move to the previous day
        else:
            break
        if streak > 365:   # safety net so it can't loop forever
            break

    print("\n--- 🔥 Weekly Spending Streak ---")
    print(f"You've stayed on budget for {streak} day(s) in a row!")
    if streak >= 7:
        print("🏆 Full week streak! You're basically a finance wizard.")
    return streak


def find_villain(data):
    """Find the category with the highest total spend — that's your villain."""
    print("\n--- 🦹 Big Money Villain ---")
    totals = {}
    for e in data["expenses"]:
        totals[e["category"]] = totals.get(e["category"], 0) + e["amount"]

    if not totals:
        print("No villain yet — you haven't logged any expenses!")
        return None, 0

    # Find the category with the max total spend
    worst_category = max(totals, key=totals.get)
    worst_amount = totals[worst_category]
    villain_name = VILLAIN_NAMES.get(worst_category, f"The {worst_category.title()} Menace 👹")

    print(f"Your arch-nemesis is... {villain_name}")
    print(f"It has drained ₹{worst_amount:.2f} from you in total.")
    return villain_name, worst_amount


def generate_roast(data, total_spent, budget, villain_name):
    """Pick a roast line based on budget status and mood patterns."""
    print("\n--- 🤖 AI Spending Roast ---")

    if villain_name is None:
        print("No data to roast yet. Log some expenses first, champ.")
        return

    if total_spent > budget:
        line = random.choice(ROASTS_OVER_BUDGET)
    else:
        line = random.choice(ROASTS_UNDER_BUDGET)
    print(line.format(villain=villain_name, amount=round(total_spent)))

    # Check mood pattern: if "stressed" is the most common mood, add an extra roast
    moods = [e["mood"] for e in data["expenses"]]
    if moods and moods.count("stressed") >= len(moods) / 2:
        print(random.choice(ROASTS_MOOD_STRESS))


def view_history(data):
    """Print every logged expense, most recent first."""
    print("\n--- 📜 Expense History ---")
    if not data["expenses"]:
        print("Nothing logged yet.")
        return
    for e in reversed(data["expenses"]):
        emoji = next((em for name, em in MOODS.values() if name == e["mood"]), "🤷")
        print(f"{e['date']} | {e['category']:<12} | ₹{e['amount']:<8.2f} | {emoji} {e['mood']}")


def set_budget(data):
    """Let the user update their weekly budget."""
    while True:
        try:
            new_budget = float(input("Enter new weekly budget (₹): "))
            data["weekly_budget"] = new_budget
            save_data(data)
            print(f"✅ Weekly budget set to ₹{new_budget:.2f}")
            break
        except ValueError:
            print("Please enter a valid number.")


def main_menu():
    data = load_data()
    print("💸 Welcome to CHAOS WALLET 💸")

    while True:
        print("\n============================")
        print("1. ➕ Add expense")
        print("2. 🩸 Show Financial Damage Meter")
        print("3. 🔥 Show Weekly Streak")
        print("4. 🦹 Reveal Big Money Villain")
        print("5. 🤖 Get Roasted")
        print("6. 📜 View history")
        print("7. ⚙️  Set weekly budget")
        print("8. 🚪 Exit")
        choice = input("Choose an option (1-8): ").strip()

        if choice == "1":
            add_expense(data)
        elif choice == "2":
            show_damage_meter(data)
        elif choice == "3":
            calculate_streak(data)
        elif choice == "4":
            find_villain(data)
        elif choice == "5":
            total_spent, budget = show_damage_meter(data)
            villain_name, _ = find_villain(data)
            generate_roast(data, total_spent, budget, villain_name)
        elif choice == "6":
            view_history(data)
        elif choice == "7":
            set_budget(data)
        elif choice == "8":
            print("👋 Stay solvent out there!")
            break
        else:
            print("Not a valid option, try 1-8.")


if __name__ == "__main__":
    main_menu()
