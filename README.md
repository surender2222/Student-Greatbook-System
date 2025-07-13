import os
from typing import Dict, Callable

import pandas as pd


class Gradebook:
    """Encapsulates all gradebook operations."""

    FILE = 'gradebook.csv'

    def __init__(self) -> None:
        if os.path.exists(self.FILE):
            # load existing records
            self.df = pd.read_csv(self.FILE)
        else:
            # create a fresh, empty gradebook
            self.df = pd.DataFrame(
                columns=["StudentID", "Name", "Subject", "Score"], dtype=str
            )
            self.save()

    # ---------------------------------------------------------------------
    # CRUD helpers
    # ---------------------------------------------------------------------
    def save(self) -> None:
        """Persist dataframe to disk."""
        self.df.to_csv(self.FILE, index=False)

    # ------------------------------------------------------------------
    # Menu actions
    # ------------------------------------------------------------------
    def add_student(self) -> None:
        sid = input("Student ID           : ").strip()
        name = input("Student name         : ").strip().title()

        if sid in self.df["StudentID"].values:
            print("❗ Student ID already exists. Use a different ID.")
            return

        # Add a placeholder row (no subject yet)
        self.df.loc[len(self.df)] = [sid, name, "", ""]
        self.save()
        print("✅ Student added successfully!")

    def add_grade(self) -> None:
        sid = input("Student ID           : ").strip()
        if sid not in self.df["StudentID"].values:
            print("❗ Student not found. Add the student first.")
            return

        subject = input("Subject              : ").strip().title()
        try:
            score = float(input("Score (0–100)        : ").strip())
            if not 0 <= score <= 100:
                raise ValueError
        except ValueError:
            print("❗ Please enter a valid number between 0 and 100.")
            return

        name = (
            self.df.loc[self.df["StudentID"] == sid, "Name"].iloc[0]
        )  # keep name consistent
        self.df.loc[len(self.df)] = [sid, name, subject, score]
        self.save()
        print("✅ Grade recorded!")

    def edit_grade(self) -> None:
        sid = input("Student ID           : ").strip()
        subject = input("Subject to update    : ").strip().title()

        mask = (self.df["StudentID"] == sid) & (self.df["Subject"] == subject)
        if mask.sum() == 0:
            print("❗ Record not found.")
            return

        try:
            new_score = float(input("New score (0–100)     : ").strip())
            if not 0 <= new_score <= 100:
                raise ValueError
        except ValueError:
            print("❗ Invalid score.")
            return

        self.df.loc[mask, "Score"] = new_score
        self.save()
        print("✅ Grade updated!")

    def delete_student(self) -> None:
        sid = input("Student ID to delete : ").strip()
        if sid not in self.df["StudentID"].values:
            print("❗ Student not found.")
            return
        self.df = self.df[self.df["StudentID"] != sid].reset_index(drop=True)
        self.save()
        print("✅ Student and all their grades deleted.")

    def view_student_report(self) -> None:
        sid = input("Student ID           : ").strip()
        records = self.df[self.df["StudentID"] == sid]
        if records.empty:
            print("❗ Student not found.")
            return

        print("\nReport Card for", records.iloc[0]["Name"])
        print("-" * 40)
        if (records["Subject"] == "").all():
            print("(No grades recorded yet)")
            return
        # Convert Score to numeric for averaging
        records = records[records["Subject"] != ""].copy()
        records["Score"] = pd.to_numeric(records["Score"])
        print(records[["Subject", "Score"]].to_string(index=False))
        print("Average Score :", records["Score"].mean().round(2))

    def view_all(self) -> None:
        if self.df.empty:
            print("Gradebook is empty.")
            return
        print(self.df.to_string(index=False))


# ----------------------------------------------------------------------
# CLI / Main loop
# ----------------------------------------------------------------------

MENU: Dict[str, Callable[[None], None]] = {}

def register_option(key: str, title: str):
    """Decorator to register a menu option."""

    def _decorator(func):
        MENU[key] = (title, func)
        return func

    return _decorator


gb = Gradebook()


@register_option("1", "Add new student")
def _( ):  # noqa: D401,E301
    gb.add_student()


@register_option("2", "Record a new grade")
def _( ):  # noqa: D401,E301
    gb.add_grade()


@register_option("3", "Edit an existing grade")
def _( ):  # noqa: D401,E301
    gb.edit_grade()


@register_option("4", "Delete a student (and their grades)")
def _( ):  # noqa: D401,E301
    gb.delete_student()


@register_option("5", "View single student report card")
def _( ):  # noqa: D401,E301
    gb.view_student_report()


@register_option("6", "View entire gradebook")
def _( ):  # noqa: D401,E301
    gb.view_all()


@register_option("0", "Exit")
def _( ):  # noqa: D401,E301
    raise SystemExit


# -----------------------
# Entry point
# -----------------------
if __name__ == "__main__":
    while True:
        print("\n====== Student Gradebook System ======")
        for k, (title, _) in sorted(MENU.items()):
            print(f"{k}. {title}")
        choice = input("Select an option ➜ ").strip()
        action = MENU.get(choice)
        if action is None:
            print("❗ Invalid choice. Try again!")
            continue
        try:
            action[1]()
        except KeyboardInterrupt:
            print("\n↩️  Cancelled. Returning to menu.")
        except SystemExit:
            print("👋 Bye! Your data are saved in 'gradebook.csv'.")
            exit()
