import json
import os
from datetime import datetime
from kivy.app import App
from kivy.clock import Clock
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.scrollview import ScrollView
from kivy.uix.label import Label
from kivy.uix.button import Button
from kivy.uix.textinput import TextInput

PROFILE_FILE = "profile.json"
HISTORY_FILE = "history.json"

EXERCISES = [
    {
        "id": "jj",
        "name": "Jumping Jacks",
        "level": "Beginner",
        "intensity": 8,  # cals/min at ~70kg
        "howTo": [
            "Stand tall with feet together and arms at your sides.",
            "Jump feet out while raising arms overhead.",
            "Jump back to start position.",
            "Keep steady rhythm and breathe normally.",
        ],
        "nextTips": [
            "Drink water and slow your breathing for 1 minute.",
            "Do a light shoulder and calf stretch.",
            "Next: try Bodyweight Squats for 1–3 minutes.",
        ],
    },
    {
        "id": "sq",
        "name": "Bodyweight Squats",
        "level": "Beginner",
        "intensity": 7,
        "howTo": [
            "Stand with feet shoulder-width apart.",
            "Push hips back and bend knees like sitting in a chair.",
            "Keep chest up and back straight.",
            "Push through heels to stand up.",
        ],
        "nextTips": [
            "Shake out legs and walk slowly for 1 minute.",
            "Stretch quads and hamstrings.",
            "Next: try Push-Ups (knees ok) for 1–2 minutes.",
        ],
    },
    {
        "id": "pu",
        "name": "Push-Ups",
        "level": "Intermediate",
        "intensity": 7.5,
        "howTo": [
            "Start in a plank with hands under shoulders.",
            "Keep body straight from head to heels.",
            "Lower chest by bending elbows.",
            "Push back up. Use knees if needed.",
        ],
        "nextTips": [
            "Rest 60 seconds and sip water.",
            "Try a 20–30 second plank hold.",
            "Next: do gentle chest/arm stretches.",
        ],
    },
]


def load_json(path, default):
    if not os.path.exists(path):
        return default
    try:
        with open(path, "r", encoding="utf-8") as f:
            return json.load(f)
    except Exception:
        return default


def save_json(path, data):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2)


def mmss(seconds: int) -> str:
    m = seconds // 60
    s = seconds % 60
    return f"{m:02d}:{s:02d}"


def day_key(iso: str) -> str:
    return datetime.fromisoformat(iso).strftime("%a %b %d %Y")


def time_only(iso: str) -> str:
    return datetime.fromisoformat(iso).strftime("%I:%M %p").lstrip("0")


class FitAppUI(BoxLayout):
    def __init__(self, **kwargs):
        super().__init__(orientation="vertical", **kwargs)

        self.profile = load_json(PROFILE_FILE, {"name": "", "weightKg": ""})
        self.history = load_json(HISTORY_FILE, [])

        self.selected = None
        self.detail_tab = "HOW"  # HOW or START

        self.timer_running = False
        self.start_iso = None
        self.elapsed = 0
        self._tick_event = None

        self.header = Label(text="Fit Tracker", size_hint_y=None, height=50, font_size=22)
        self.add_widget(self.header)

        self.body = BoxLayout(orientation="vertical")
        self.add_widget(self.body)

        self.go_home()

    # ---------- navigation helpers ----------
    def set_title(self, t):
        self.header.text = t

    def clear_body(self):
        self.body.clear_widgets()

    def add_back(self, target_fn):
        btn = Button(text="Back", size_hint_y=None, height=44)
        btn.bind(on_press=lambda *_: target_fn())
        self.body.add_widget(btn)

    # ---------- screens ----------
    def go_home(self):
        self.set_title("Fit Tracker")
        self.clear_body()

        card = BoxLayout(orientation="vertical", padding=10, spacing=8)
        name = self.profile.get("name") or "(not set)"
        w = self.profile.get("weightKg") or "(not set)"
        card.add_widget(Label(text=f"Name: {name}", halign="left"))
        card.add_widget(Label(text=f"Weight (kg): {w}", halign="left"))

        b1 = Button(text="Edit Profile", size_hint_y=None, height=44)
        b1.bind(on_press=lambda *_: self.go_profile())

        b2 = Button(text="Choose Exercise", size_hint_y=None, height=44)
        b2.bind(on_press=lambda *_: self.go_list())

        b3 = Button(text="Calendar / History", size_hint_y=None, height=44)
        b3.bind(on_press=lambda *_: self.go_history())

        card.add_widget(b1)
        card.add_widget(b2)
        card.add_widget(b3)

        self.body.add_widget(card)

    def go_profile(self):
        self.set_title("Profile")
        self.clear_body()

        box = BoxLayout(orientation="vertical", padding=10, spacing=8)

        box.add_widget(Label(text="Name", size_hint_y=None, height=24))
        name_in = TextInput(text=self.profile.get("name", ""), multiline=False, size_hint_y=None, height=44)

        box.add_widget(Label(text="Weight (kg)", size_hint_y=None, height=24))
        weight_in = TextInput(text=str(self.profile.get("weightKg", "")), multiline=False, input_filter="float",
                              size_hint_y=None, height=44)

        def save_profile(_):
            self.profile["name"] = name_in.text.strip()
            self.profile["weightKg"] = weight_in.text.strip()
            save_json(PROFILE_FILE, self.profile)
            self.go_home()

        save_btn = Button(text="Save", size_hint_y=None, height=44)
        save_btn.bind(on_press=save_profile)

        box.add_widget(name_in)
        box.add_widget(weight_in)
        box.add_widget(save_btn)

        self.body.add_widget(box)
        self.add_back(self.go_home)

    def go_list(self):
        self.set_title("Exercises")
        self.clear_body()

        scroll = ScrollView()
        cont = BoxLayout(orientation="vertical", padding=10, spacing=8, size_hint_y=None)
        cont.bind(minimum_height=cont.setter("height"))

        for ex in EXERCISES:
            btn = Button(text=f"{ex['name']}  ({ex['level']})", size_hint_y=None, height=50)

            def make_open(e):
                return lambda *_: self.go_detail(e)

            btn.bind(on_press=make_open(ex))
            cont.add_widget(btn)

        scroll.add_widget(cont)
        self.body.add_widget(scroll)
        self.add_back(self.go_home)

    def go_detail(self, ex):
        self.selected = ex
        self.detail_tab = "HOW"
        self.set_title(ex["name"])
        self.clear_body()

        # tab buttons
        tabrow = BoxLayout(orientation="horizontal", size_hint_y=None, height=44, spacing=8, padding=10)
        how_btn = Button(text="How to do")
        start_btn = Button(text="Start workout")

        def set_tab(tab):
            self.detail_tab = tab
            self.go_detail(self.selected)

        how_btn.bind(on_press=lambda *_: set_tab("HOW"))
        start_btn.bind(on_press=lambda *_: set_tab("START"))

        tabrow.add_widget(how_btn)
        tabrow.add_widget(start_btn)
        self.body.add_widget(tabrow)

        card = BoxLayout(orientation="vertical", padding=10, spacing=8)

        if self.detail_tab == "HOW":
            txt = "\n".join([f"{i+1}. {s}" for i, s in enumerate(ex["howTo"])])
            card.add_widget(Label(text=txt, halign="left", valign="top"))
        else:
            card.add_widget(Label(text="Ready? Press Start to begin the timer.", halign="left"))
            card.add_widget(Label(text="Calories estimate uses your weight and intensity.", halign="left"))
            b = Button(text="Start", size_hint_y=None, height=44)
            b.bind(on_press=lambda *_: self.start_workout())
            card.add_widget(b)

        self.body.add_widget(card)
        self.add_back(self.go_list)

    def start_workout(self):
        if not self.selected:
            return

        try:
            w = float(self.profile.get("weightKg") or "0")
        except ValueError:
            w = 0

        if w <= 0:
            self.set_title("Profile needed")
            self.clear_body()
            self.body.add_widget(Label(text="Please set your weight in Profile first."))
            self.add_back(self.go_profile)
            return

        self.timer_running = True
        self.start_iso = datetime.now().isoformat()
        self.elapsed = 0

        if self._tick_event:
            self._tick_event.cancel()
        self._tick_event = Clock.schedule_interval(self._tick, 1)

        self.go_workout()

    def _tick(self, _dt):
        if self.timer_running:
            self.elapsed += 1

    def calc_calories(self):
        if not self.selected:
            return 0.0
        try:
            w = float(self.profile.get("weightKg") or "70")
        except ValueError:
            w = 70.0
        factor = w / 70.0
        minutes = self.elapsed / 60.0
        return minutes * float(self.selected["intensity"]) * factor

    def go_workout(self):
        if not self.selected:
            self.go_home()
            return

        self.set_title("Workout")
        self.clear_body()

        card = BoxLayout(orientation="vertical", padding=10, spacing=8)
        card.add_widget(Label(text=f"Exercise: {self.selected['name']}", font_size=18))

        timer_lbl = Label(text=f"Time: {mmss(self.elapsed)}", font_size=22)
        cal_lbl = Label(text=f"Calories: {self.calc_calories():.1f}", font_size=18)

        def refresh_ui(_dt):
            timer_lbl.text = f"Time: {mmss(self.elapsed)}"
            cal_lbl.text = f"Calories: {self.calc_calories():.1f}"

        Clock.schedule_interval(refresh_ui, 0.5)

        stop_btn = Button(text="Stop & Save", size_hint_y=None, height=44)
        stop_btn.bind(on_press=lambda *_: self.stop_workout())

        card.add_widget(timer_lbl)
        card.add_widget(cal_lbl)
        card.add_widget(stop_btn)

        self.body.add_widget(card)

    def stop_workout(self):
        if not self.selected or not self.start_iso:
            return

        self.timer_running = False
        if self._tick_event:
            self._tick_event.cancel()
            self._tick_event = None

        end_iso = datetime.now().isoformat()
        duration_min = self.elapsed / 60.0
        calories = self.calc_calories()

        session = {
            "id": str(int(datetime.now().timestamp() * 1000)),
            "exerciseId": self.selected["id"],
            "exerciseName": self.selected["name"],
            "startIso": self.start_iso,
            "endIso": end_iso,
            "durationMin": duration_min,
            "calories": calories,
        }

        self.history.insert(0, session)
        save_json(HISTORY_FILE, self.history)

        self.elapsed = 0
        self.start_iso = None

        self.go_history(show_tips_for=self.selected)

    def go_history(self, show_tips_for=None):
        self.set_title("Calendar / History")
        self.clear_body()

        # clear history button
        clear_btn = Button(text="Clear History", size_hint_y=None, height=44)

        def do_clear(_):
            self.history = []
            save_json(HISTORY_FILE, self.history)
            self.go_history()

        clear_btn.bind(on_press=do_clear)
        self.body.add_widget(clear_btn)

        if not self.history:
            self.body.add_widget(Label(text="No workouts yet. Start one from Exercises."))
            self.add_back(self.go_home)
            return

        # optional tips after stop
        if show_tips_for is not None:
            tipbox = BoxLayout(orientation="vertical", padding=10, spacing=4, size_hint_y=None)
            tipbox.bind(minimum_height=tipbox.setter("height"))
            tipbox.add_widget(Label(text="Tips for what to do next:", bold=True))
            for t in show_tips_for.get("nextTips", [])[:3]:
                tipbox.add_widget(Label(text=f"• {t}", halign="left"))
            self.body.add_widget(tipbox)

        # group by day
        groups = {}
        for s in self.history:
            k = day_key(s["startIso"])
            groups.setdefault(k, []).append(s)

        scroll = ScrollView()
        cont = BoxLayout(orientation="vertical", padding=10, spacing=10, size_hint_y=None)
        cont.bind(minimum_height=cont.setter("height"))

        for day in groups.keys():
            cont.add_widget(Label(text=day, font_size=16, bold=True, size_hint_y=None, height=30))
            for s in groups[day]:
                box = BoxLayout(orientation="vertical", padding=8, spacing=2, size_hint_y=None)
                box.bind(minimum_height=box.setter("height"))
                box.add_widget(Label(text=s["exerciseName"], bold=True))
                box.add_widget(Label(text=f"Start: {time_only(s['startIso'])}   End: {time_only(s['endIso'])}"))
                box.add_widget(Label(text=f"Duration: {s['durationMin']:.1f} min   Calories: {s['calories']:.1f}"))
                cont.add_widget(box)

        scroll.add_widget(cont)
        self.body.add_widget(scroll)

        self.add_back(self.go_home)


class FitTrackerApp(App):
    def build(self):
        return FitAppUI()


if __name__ == "__main__":
    FitTrackerApp().run()
