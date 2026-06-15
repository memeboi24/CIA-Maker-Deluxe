import os
import random
import re
import shutil
import subprocess
import wave
import webbrowser
import tkinter as tk
from tkinter import filedialog, messagebox, ttk
from tkinter.scrolledtext import ScrolledText

try:
    from PIL import Image, ImageTk
except ImportError:
    Image = None
    ImageTk = None


# -----------------------------
# CONFIG
# -----------------------------
BASE_DIR = os.path.abspath(os.path.dirname(__file__))
OUTPUT_DIR = os.path.join(BASE_DIR, "output")

MAKEROM_PATH = "./makerom"
BANNRTOOL_PATH = "./bannertool"
DEFAULT_RSF = os.path.join(BASE_DIR, "standard.rsf")

# Image and audio size requirements
AUDIO_LENGHT = 3.0
BANNER_SIZE = (256, 128)
ICON_SIZE = (48, 48)
SMALL_ICON_SIZE = (24, 24)


def ensure_output_dir():
    os.makedirs(OUTPUT_DIR, exist_ok=True)


def abs_path(path: str) -> str:
    return os.path.abspath(os.path.expanduser(path.strip()))


def file_exists(path: str) -> bool:
    return bool(path) and os.path.isfile(path)


def validate_png_size(path: str, expected_size: tuple[int, int]) -> tuple[bool, str]:
    if Image is None:
        return False, "Pillow is required for image validation & preview. Install it with: pip install pillow"

    if not file_exists(path):
        return False, f"Missing file: {path}"

    try:
        with Image.open(path) as img:
            if img.mode not in ("RGB", "RGBA"):
                img = img.convert("RGBA")
            if img.size != expected_size:
                return False, f"{os.path.basename(path)} must be exactly {expected_size[0]}x{expected_size[1]} (got {img.size[0]}x{img.size[1]})"
    except Exception as exc:
        return False, f"Could not open {path}: {exc}"

    return True, "OK"

def validate_wav_lenght(file_path, max_duration=3.0):
    try:
        with wave.open(file_path, "r") as audio_file:
            frame_rate = audio_file.getframerate()
            n_frames = audio_file.getnframes()
            duration = n_frames / float(frame_rate)

        if duration > max_duration:
            return (
                False,
                f"Audio is {duration:.2f} seconds long.\n"
                f"3DS banner audio must be {max_duration:.0f} seconds or shorter."
            )

        return True, "OK"

    except Exception as exc:
        return False, f"Failed to read WAV file:\n{exc}"


def preview_image(path: str, max_size: tuple[int, int]) -> "ImageTk.PhotoImage | None":
    if Image is None or ImageTk is None or not file_exists(path):
        return None
    try:
        with Image.open(path) as img:
            img = img.convert("RGBA")
            img.thumbnail(max_size)
            return ImageTk.PhotoImage(img)
    except Exception:
        return None


def run_cmd(cmd: list[str], log_fn):
    log_fn("$ " + " ".join(cmd))
    proc = subprocess.run(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
        text=True
    )
    if proc.stdout:
        for line in proc.stdout.splitlines():
            log_fn(line)
    if proc.returncode != 0:
        raise subprocess.CalledProcessError(proc.returncode, cmd, output=proc.stdout)
    return proc


def generate_title_id() -> str:
    webbrowser.open('https://studionamehere.github.io/HomebrewTitleIDGenerator/')


def normalize_title_id(value: str) -> str:
    raw = value.strip().upper()
    if raw.startswith("0X"):
        raw = raw[2:]
    if not raw:
        return ""
    if not re.fullmatch(r"[0-9A-F]{1,16}", raw):
        raise ValueError("Title ID must be 1 to 16 hexadecimal characters.")
    return raw.zfill(16)


class WizardApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("CIA Maker Deluxe Wizard")
        self.geometry("980x640")
        self.minsize(1000, 800)
        self.current_step = 0

        self.banner_bnr = os.path.join(OUTPUT_DIR, "banner.bnr")
        self.banner_bin = os.path.join(OUTPUT_DIR, "banner.bin")
        self.smdh_out = os.path.join(OUTPUT_DIR, "app.smdh")
        self.cia_out = os.path.join(OUTPUT_DIR, "output.cia")

        self.toggle_log_btn_visible = True

        self.vars = {
            "title": tk.StringVar(value="My App"),
            "description": tk.StringVar(value="My App"),
            "author": tk.StringVar(value="Author"),
            "title_id": tk.StringVar(value=""),
            "audio": tk.StringVar(),
            "banner": tk.StringVar(),
            "icon": tk.StringVar(),
            "icon_small": tk.StringVar(),
            "elf": tk.StringVar(),
            "rsf": tk.StringVar(value=DEFAULT_RSF),
            "cia_name": tk.StringVar(value="output"),
        }

        self.previews = {
            "banner": None,
            "icon": None,
            "icon_small": None,
        }

        self._build_ui()

    # -----------------------------
    # UI
    # -----------------------------
    def _build_ui(self):
        self.columnconfigure(0, weight=1)
        self.rowconfigure(1, weight=1)

        top = ttk.Frame(self, padding=(18, 12, 18, 6))
        top.grid(row=0, column=0, sticky="ew")
        top.columnconfigure(0, weight=1)

        self.progress = tk.Canvas(top, height=90, highlightthickness=0)
        self.progress.grid(row=0, column=0, sticky="ew")
        self.progress.bind("<Configure>", lambda e: self.draw_progress())

        self.container = ttk.Frame(self, padding=(18, 6, 18, 12))
        self.container.grid(row=1, column=0, sticky="nsew")
        self.container.columnconfigure(0, weight=1)
        self.container.rowconfigure(0, weight=1)

        self.pages = []
        self.pages.append(self._build_step1_page(self.container))
        self.pages.append(self._build_step2_page(self.container))
        self.pages.append(self._build_step3_page(self.container))

        for page in self.pages:
            page.grid(row=0, column=0, sticky="nsew")

        bottom = ttk.Frame(self, padding=(18, 0, 18, 18))
        bottom.grid(row=2, column=0, sticky="ew")
        bottom.columnconfigure(0, weight=1)

        self.back_btn = ttk.Button(bottom, text="Back", command=self.go_back)
        self.back_btn.grid(row=0, column=0, sticky="w")

        self.next_btn = ttk.Button(bottom, text="Continue", command=self.go_next)
        self.next_btn.grid(row=0, column=1, sticky="e")

        self.toggle_log_btn = ttk.Button(bottom, text="Toggle Log", command=self.toggle_log)
        self.toggle_log_btn.grid(row=0, column=0, sticky="se")
        self.toggle_log_btn.configure(state="normal")
        
        self.log_frame = ttk.LabelFrame(bottom, text="Log", padding=8)
        self.log_frame.grid(row=1, column=0, columnspan=2, sticky="ew", pady=(10, 0))
        self.log_frame.columnconfigure(0, weight=1)

        self.log = ScrolledText(self.log_frame, height=10, wrap="word")
        self.log.grid(row=0, column=0, sticky="ew")
        self.log.configure(state="disabled")

        self.show_step(0)

    def toggle_log(self):
        if self.toggle_log_btn_visible == True:
              self.log_frame.grid_remove()
              self.toggle_log_btn_visible = False
        else:
              self.log_frame.grid()
              self.toggle_log_btn_visible = True

    def draw_progress(self):
        c = self.progress
        c.delete("all")

        w = max(c.winfo_width(), 1)
        h = max(c.winfo_height(), 1)
        y = 38

        xs = [w * 0.30, w * 0.50, w * 0.70]

        def fill(i):
            return "#FFA500" if i <= self.current_step else "#5a5a5a"

        def outline(i):
            return "#FFA500" if i <= self.current_step else "#5a5a5a"

        c.create_line(xs[0], y, xs[1], y, width=6, fill=outline(0))
        c.create_line(xs[1], y, xs[2], y, width=6, fill=outline(1))

        labels = ["Metadata", "Banner", "CIA"]
        for i, x in enumerate(xs):
            r = 16
            c.create_oval(x-r, y-r, x+r, y+r, fill=fill(i), outline=outline(i), width=2)
            c.create_text(x, y + 28, text=labels[i], font=("Segoe UI", 10, "bold"), fill="#222")

        c.create_text(w * 0.1, 7, text="CIA Setup Wizard", font=("Segoe UI", 16, "bold"), fill="#111")

    def show_step(self, step_index: int):
        self.current_step = step_index
        self.draw_progress()

        for i, page in enumerate(self.pages):
            if i == step_index:
                page.tkraise()

        self.back_btn.configure(state=("disabled" if step_index == 0 else "normal"))
        self.next_btn.configure(text=("Build CIA & Finish" if step_index == len(self.pages) - 1 else "Continue"))

    def go_back(self):
        if self.current_step > 0:
            self.show_step(self.current_step - 1)

    def go_next(self):
        if self.current_step == 0:
            self.run_step1()
            self.show_step(1)
        elif self.current_step == 1:
            self.run_step2()
            self.show_step(2)
        else:
            self.run_step3()

    def log_line(self, text: str):
        self.log.configure(state="normal")
        self.log.insert("end", text + "\n")
        self.log.see("end")
        self.log.configure(state="disabled")
        self.update_idletasks()

    # -----------------------------
    # Page 1
    # -----------------------------
    def _build_step1_page(self, parent):
        page = ttk.Frame(parent)
        page.columnconfigure(0, weight=1)
        page.columnconfigure(1, weight=1)
        page.rowconfigure(0, weight=1)

        left = ttk.LabelFrame(page, text="Step 1: Metadata", padding=12)
        left.grid(row=0, column=0, sticky="nsew", padx=(0, 8))
        left.columnconfigure(1, weight=1)

        right = ttk.LabelFrame(page, text="Guide", padding=12)
        right.grid(row=0, column=1, sticky="nsew", padx=(8, 0))
        right.columnconfigure(0, weight=1)

        note = ttk.Label(
            left,
            text="Enter the app title, description, author and the title id here",
            wraplength=360
        )
        note.grid(row=0, column=0, columnspan=3, sticky="w", pady=(0, 10))

        ttk.Label(left, text="Title").grid(row=1, column=0, sticky="w", pady=4)
        ttk.Entry(left, textvariable=self.vars["title"]).grid(row=1, column=1, columnspan=2, sticky="ew", pady=4)

        ttk.Label(left, text="Description").grid(row=2, column=0, sticky="w", pady=4)
        ttk.Entry(left, textvariable=self.vars["description"]).grid(row=2, column=1, columnspan=2, sticky="ew", pady=4)

        ttk.Label(left, text="Author(s)").grid(row=3, column=0, sticky="w", pady=4)
        ttk.Entry(left, textvariable=self.vars["author"]).grid(row=3, column=1, columnspan=2, sticky="ew", pady=4)

        ttk.Label(left, text="Title ID").grid(row=4, column=0, sticky="w", pady=4)
        ttk.Entry(left, textvariable=self.vars["title_id"]).grid(row=4, column=1, sticky="ew", pady=4, padx=(6, 6))
        ttk.Button(left, text="Go to Title ID Randomizer (opens browser)", command=generate_title_id).grid(row=4, column=2, sticky="e", pady=4)

        ttk.Label(
            right,
            text="Welcome to the CIA Maker Deluxe Wizard! This first here step will guide you through making the metadata required for the cia",
            wraplength=320
        ).grid(row=0, column=0, sticky="nw", pady=(0, 10))

        ttk.Label(
            right,
            text="If a title ID is supplied, the wizard will try to validate so see if it is usable, (please note that this is a work in progress and probably doesnt work)",
            wraplength=320
        ).grid(row=1, column=0, sticky="nw", pady=(0, 10))

        ttk.Label(
            right,
            text="When you are ready and have filled out the required metadata, you may proceed to the next step after the wizard has made the nessecary files",
            wraplength=320
        ).grid(row=2, column=0, sticky="nw")
        return page

    def _build_step2_page(self, parent):
        page = ttk.Frame(parent)
        page.columnconfigure(0, weight=1)
        page.columnconfigure(1, weight=1)
        page.rowconfigure(0, weight=1)

        left = ttk.LabelFrame(page, text="Step 2: Banner & Icon Assets", padding=12)
        left.grid(row=0, column=0, sticky="nsew", padx=(0, 8))
        left.columnconfigure(1, weight=1)

        right = ttk.LabelFrame(page, text="Previews", padding=12)
        right.grid(row=0, column=1, sticky="nsew", padx=(8, 0))
        right.columnconfigure(0, weight=1)

        note = ttk.Label(
            left,
            text="The Banner PNG must be exactly 256x128, The App icon must be 48x48 and The Small icon must be 24x24.",
            wraplength=360
        )
        note.grid(row=0, column=0, columnspan=3, sticky="w", pady=(0, 10))

        self._field(left, "Wav file", "audio", 1, types=[("AUDIO Files", "*.wav")])
        self._field(left, "Banner image", "banner", 2, types=[("PNG files", "*.png")])
        self._field(left, "app icon", "icon", 3, types=[("PNG files", "*.png")])
        self._field(left, "small app icon", "icon_small", 4, types=[("PNG files", "*.png")])
        
        right = ttk.LabelFrame(page, text="Guide & Previews", padding=12)
        right.grid(row=0, column=1, sticky="nsew", padx=(8, 0))
        right.columnconfigure(1, weight=1)
        ttk.Label(
            right,
            text="And now you're done with metadata, in this step you will provide the nessecary images and the audio for the jingle that will play when the app is hovered over, there are preview images beside this text if you have filled out the form to make it easier to see if you like how the banner looks",
            wraplength=320
        ).grid(row=0, column=2, sticky="nw", pady=(0, 10))

        self.banner_preview_lbl = ttk.Label(right, text="Banner preview\n(256x128)", anchor="center")
        self.banner_preview_lbl.grid(row=0, column=0, sticky="ew", pady=(0, 10))

        self.icon_preview_lbl = ttk.Label(right, text="Icon preview\n(48x48)", anchor="center")
        self.icon_preview_lbl.grid(row=1, column=0, sticky="ew", pady=(0, 10))

        self.small_icon_preview_lbl = ttk.Label(right, text="Small icon preview\n(24x24)", anchor="center")
        self.small_icon_preview_lbl.grid(row=2, column=0, sticky="ew", pady=(0, 10))

        return page

    def _field(self, parent, label, key, row, types=None):
        ttk.Label(parent, text=label).grid(row=row, column=0, sticky="w", pady=4)
        entry = ttk.Entry(parent, textvariable=self.vars[key])
        entry.grid(row=row, column=1, sticky="ew", pady=4, padx=(6, 6))
        ttk.Button(parent, text="Browse", command=lambda: self.pick_file(key, types)).grid(row=row, column=2, sticky="e", pady=4)

    def pick_file(self, key, types=None):
        path = filedialog.askopenfilename(filetypes=types or [("All files", "*.*")])
        if path:
            self.vars[key].set(path)
            self.refresh_previews()

    def refresh_previews(self):
        banner = self.vars["banner"].get().strip()
        icon = self.vars["icon"].get().strip()
        small_icon = self.vars["icon_small"].get().strip()

        self.previews["banner"] = preview_image(banner, (360, 180))
        self.previews["icon"] = preview_image(icon, (128, 128))
        self.previews["icon_small"] = preview_image(small_icon, (96, 96))

        if self.previews["banner"] is not None:
            self.banner_preview_lbl.configure(image=self.previews["banner"], text="")
        else:
            self.banner_preview_lbl.configure(image="", text="Banner preview\n(256x128)")

        if self.previews["icon"] is not None:
            self.icon_preview_lbl.configure(image=self.previews["icon"], text="")
        else:
            self.icon_preview_lbl.configure(image="", text="Icon preview\n(48x48)")

        if self.previews["icon_small"] is not None:
            self.small_icon_preview_lbl.configure(image=self.previews["icon_small"], text="")
        else:
            self.small_icon_preview_lbl.configure(image="", text="Small icon preview\n(24x24)")

    def randomize_title_id(self): #unused for now
        title_id = generate_title_id()
        self.vars["title_id"].set(title_id)
        self.log_line(f"title ID: {title_id}")

    # -----------------------------
    # Page 3
    # -----------------------------
    def _build_step3_page(self, parent):
        page = ttk.Frame(parent)
        page.columnconfigure(0, weight=1)

        box = ttk.LabelFrame(page, text="Final Step: Build CIA", padding=12)
        box.grid(row=0, column=0, sticky="nsew")
        box.columnconfigure(1, weight=1)

        right = ttk.LabelFrame(page, text="Guide", padding=12)
        right.grid(row=0, column=1, sticky="nsew", padx=(8, 0))
        right.columnconfigure(1, weight=1)
        ttk.Label(
            right,
            text="Okay!! now you're at the final step, in the final step here, you are gonna give the .elf file that got built beside your 3dsx file, the smdh and the banner file is filled out for you so you dont have to point to the output folder, just because im nice :D, oh and the rsf file path shouldnt be modified because the file can be used on pretty much all homebrew apps, if you have a specific one you need to use then you can point to that instead, Final note: the cia will be saved to the ''output'' folder",
            wraplength=320
        ).grid(row=5, column=1, sticky="nw", pady=(0, 10))

        self._field_box(box, ".ELF file", "elf", 0, types=[("ELF files", "*.elf"), ("All files", "*.*")])
        self._field_box(box, ".SMDH file", None, 1, fixed_value=self.smdh_out)
        self._field_box(box, "Banner file", None, 2, fixed_value=self.banner_bnr)
        self._field_box(box, ".RSF file", "rsf", 3, types=[("RSF files", "*.rsf"), ("All files", "*.*")])

        ttk.Label(box, text="Cia name").grid(row=4, column=0, sticky="w", pady=4)
        ttk.Entry(box, textvariable=self.vars["cia_name"]).grid(row=4, column=1, sticky="ew", pady=4, padx=(6, 6))
        ttk.Label(box, text=".cia").grid(row=4, column=2, sticky="w", pady=4)
        return page

    def _field_box(self, parent, label, key, row, fixed_value=None, types=None):
        ttk.Label(parent, text=label).grid(row=row, column=0, sticky="w", pady=4)
        if fixed_value is None:
            ttk.Entry(parent, textvariable=self.vars[key]).grid(row=row, column=1, sticky="ew", pady=4, padx=(6, 6))
            ttk.Button(parent, text="Browse", command=lambda: self.pick_file(key, types)).grid(row=row, column=2, sticky="e", pady=4)
        else:
            value = ttk.Entry(parent)
            value.grid(row=row, column=1, sticky="ew", pady=4, padx=(6, 6))
            value.insert(0, fixed_value)
            ttk.Label(parent, text="Automatically filled out :)").grid(row=row, column=2, sticky="w", pady=4)
            

    # -----------------------------
    # Steps
    # -----------------------------
    def validate_step1(self) -> bool:
        title = self.vars["title"].get().strip()
        description = self.vars["description"].get().strip()
        author = self.vars["author"].get().strip()
        title_id = self.vars["title_id"].get().strip()

        if not title:
            messagebox.showerror("Missing title", "Please enter a title.")
            return False
        if not description:
            messagebox.showerror("Missing description", "Please enter a description.")
            return False
        if not author:
            messagebox.showerror("Missing author", "Please enter an author.")
            return False

        if title_id:
            try:
                title_id = normalize_title_id(title_id)
            except ValueError as exc:
                messagebox.showerror("Invalid title ID", str(exc))
                return False
            self.vars["title_id"].set(title_id)

        return True

    def validate_step2(self) -> bool:
        audio = abs_path(self.vars["audio"].get())
        banner = abs_path(self.vars["banner"].get())
        icon = abs_path(self.vars["icon"].get())
        icon_small = abs_path(self.vars["icon_small"].get())

        for path in (audio, banner, icon, icon_small):
            if not file_exists(path):
                messagebox.showerror("Missing file", f"File not found:\n{path}")
                return False

        ok, msg = validate_wav_lenght(audio, 3.0)

        if not ok:
            messagebox.showerror("Audio error", msg)
            return False

        ok, msg = validate_png_size(banner, BANNER_SIZE)
        if not ok:
            messagebox.showerror("Banner image error", msg)
            return False

        ok, msg = validate_png_size(icon, ICON_SIZE)
        if not ok:
            messagebox.showerror("Icon image error", msg)
            return False

        ok, msg = validate_png_size(icon_small, SMALL_ICON_SIZE)
        if not ok:
            messagebox.showerror("Small icon image error", msg)
            return False

        return True

    def run_step1(self):
        if not self.validate_step1():
            raise RuntimeError("Step 1 failed validation.")
        self.log_line("=== Step 1: metadata saved ===")
        self.log_line(f"Title: {self.vars['title'].get().strip()}")
        self.log_line(f"Description: {self.vars['description'].get().strip()}")
        self.log_line(f"Author: {self.vars['author'].get().strip()}")
        title_id = self.vars['title_id'].get().strip()
        if title_id:
            self.log_line(f"Title ID: {title_id}")
        else:
            self.log_line("Title ID: (not set)")
        self.log_line("Step 1 finished successfully.")

    def _build_smdh_cmd(self, title_id_flag=None):
        title = self.vars["title"].get().strip()
        description = self.vars["description"].get().strip()
        author = self.vars["author"].get().strip()
        icon = abs_path(self.vars["icon"].get())
        cmd = [
            BANNRTOOL_PATH,
            "makesmdh",
            "-s", title,
            "-l", description,
            "-p", author,
        ]
        if title_id_flag:
            cmd.extend(title_id_flag)
        cmd.extend([
            "-i", icon,
            "-o", self.smdh_out,
        ])
        return cmd

    def run_step2(self):
        ensure_output_dir()
        if not self.validate_step2():
            raise RuntimeError("Step 2 failed validation.")

        audio = abs_path(self.vars["audio"].get())
        banner = abs_path(self.vars["banner"].get())
        title_id = self.vars["title_id"].get().strip()

        self.log_line("=== Step 2: build banner / SMDH assets ===")

        run_cmd(
            [
                BANNRTOOL_PATH,
                "makebanner",
                "-i", banner,
                "-a", audio,
                "-o", self.banner_bnr,
            ],
            self.log_line,
        )

        try:
            shutil.copy2(self.banner_bnr, self.banner_bin)
            self.log_line(f"Copied banner output to {self.banner_bin}")
        except Exception as exc:
            self.log_line(f"Could not create banner.bin copy: {exc}")

        smdh_flags = []
        if title_id:
            title_id = normalize_title_id(title_id)
            self.vars["title_id"].set(title_id)
            smdh_flags = [
                ["--title-id", title_id],
                ["--titleid", title_id],
                ["-t", title_id],
                ["-T", title_id],
            ]

        if smdh_flags:
            last_exc = None
            for flag in smdh_flags:
                try:
                    run_cmd(self._build_smdh_cmd(flag), self.log_line)
                    last_exc = None
                    break
                except subprocess.CalledProcessError as exc:
                    last_exc = exc
                    self.log_line(f"SMDH command rejected title ID flag {flag[0]}: trying the next form.")
            if last_exc is not None:
                self.log_line("Step 2 SMDH command failed with every title ID flag form.")
                raise last_exc
        else:
            run_cmd(self._build_smdh_cmd(), self.log_line)

        self.log_line("Step 2 finished successfully.")

    def run_step3(self):
        ensure_output_dir()

        elf = abs_path(self.vars["elf"].get())
        rsf = abs_path(self.vars["rsf"].get())

        if not file_exists(elf):
            messagebox.showerror("Missing ELF", f"ELF file not found:\n{elf}")
            return

        if not file_exists(rsf):
            messagebox.showerror("Missing RSF", f"RSF file not found:\n{rsf}")
            return

        if not file_exists(self.smdh_out):
            messagebox.showerror("Missing SMDH", f"Expected output not found:\n{self.smdh_out}")
            return

        if not file_exists(self.banner_bnr):
            messagebox.showerror("Missing banner", f"Expected output not found:\n{self.banner_bnr}")
            return

        cia_name = self.vars["cia_name"].get().strip() or "output"
        cia_out = os.path.join(OUTPUT_DIR, f"{cia_name}.cia")

        self.log_line("=== Step 3: build CIA ===")
        cmd = [
            MAKEROM_PATH,
            "-f", "cia",
            "-o", cia_out,
            "-elf", elf,
            "-rsf", rsf,
            "-icon", self.smdh_out,
            "-banner", self.banner_bnr,
            "-major", "1",
            "-minor", "0",
            "-micro", "0",
        ]

        try:
            run_cmd(cmd, self.log_line)
            self.log_line(f"Built CIA: {cia_out}")
            messagebox.showinfo("Success!!!", f"CIA created at:\n{cia_out}")
        except Exception as exc:
            messagebox.showerror("Build failed :(", str(exc))


if __name__ == "__main__":
    app = WizardApp()
    app.mainloop()
