# Playwright Automated Job Applier

## 📜 Overview
This project automates the “Easy Apply” workflow on LinkedIn using Playwright. It streamlines repetitive job‑application steps so I can target more openings in less time.

## 🛠️ Role & Tools
- **Role:** Programmer, College Graduate  
- **Tools:** Playwright, Python, VS Code, OpenAI

## 🎯 Objective
Many positions align with my cybersecurity and IT career goals, but applying individually is time‑consuming. This script automates the process so I can focus on tailoring follow‑ups and interview prep.

## ✅ Key Deliverables
- Headless Python script that logs in, filters jobs, and submits applications
- Configurable job‑level and keyword filters for precise targeting
- Re‑usable Playwright framework with clear setup instructions

## 🧠 Lessons Learned
Building this solution sharpened my browser‑automation skills, session‑management tactics, and understanding of LinkedIn’s DOM structure. It also reinforced ethical automation practices, including rate‑limiting and respectful platform use.

## 📌 Files
- `"""
LinkedIn Easy Apply Auto‑Apply Script (No Cover Letter) — July 2025 Robust Version
================================================================================
• Works with current LinkedIn markup
• Compatible with older Playwright versions (no `timeout=` on `is_visible()`)
• Targets ONLY the job‑details Easy‑Apply button (avoids clicking the search‑page
  filter pill that also says “Easy Apply”).
"""

import time
import random
from pathlib import Path
from playwright.sync_api import sync_playwright, TimeoutError as PlaywrightTimeout

# ------------------------- CONFIG -------------------------
EMAIL        = "EMAIL"
PASSWORD     = "PASSWORD"
RESUME_PATH  = "C:/FILE/PATH/TO/YOUR/RESUME.pdf"
KEYWORDS     = "Job Title OR Keywords"
LOCATION     = "United States"
PAGES        = 3
HEADLESS     = False          # True = run headless
BASE_DELAY   = (1.0, 2.2)     # Min / max wait between actions (sec)
# ----------------------------------------------------------

if RESUME_PATH and not Path(RESUME_PATH).is_file():
    raise SystemExit(f"ERROR: RESUME_PATH '{RESUME_PATH}' not found.")


def human_sleep(a: float = BASE_DELAY[0], b: float = BASE_DELAY[1]):
    time.sleep(random.uniform(a, b))


# -------------------------- LOGIN -------------------------

def login(page):
    page.goto("https://www.linkedin.com/login", timeout=30000)
    page.fill('input[id="username"]', EMAIL)
    page.fill('input[id="password"]', PASSWORD)
    page.click('button[type="submit"]')
    try:
        page.wait_for_selector('input[placeholder="Search"]', timeout=20000)
        print("✔ Logged in")
    except PlaywrightTimeout:
        raise SystemExit("❌ Login failed or timed out — check credentials / 2FA.")


# -------------------- EASY‑APPLY ROUTINE ------------------

def apply_to_job(page) -> bool:
    """Click Easy Apply within the job‑details panel and walk the wizard."""

    # Selector limited to the right‑side details panel — prevents clicking the
    # blue “Easy Apply” filter pill in the search header.
    APPLY_BTN_SEL = (
        'div.jobs-details__main-content button.jobs-apply-button,'
        'div.jobs-details__main-content button#jobs-apply-button-id'
    )

    try:
        page.wait_for_selector(APPLY_BTN_SEL, timeout=4000)
        btn = page.query_selector(APPLY_BTN_SEL)
        if not btn:
            return False
        btn.scroll_into_view_if_needed()
        btn.click()
    except PlaywrightTimeout:
        return False
    except Exception as exc:
        print(f"    → Error clicking Easy Apply: {exc}")
        return False

    human_sleep()

    # Upload résumé if a file‑input shows up (older Playwright: no timeout kw)
    try:
        page.wait_for_selector('input[type="file"]', timeout=1500)
        if RESUME_PATH:
            page.set_input_files('input[type="file"]', RESUME_PATH)
    except PlaywrightTimeout:
        pass

    # ---------------- Wizard navigation ------------------
    while True:
        human_sleep(0.6, 1.3)

        # --- final submit step ---
        if page.query_selector('button:has-text("Submit application")'):
            page.click('button:has-text("Submit application")')
            # Retry to find and click "Done" or "Close" button after submitting
            for _ in range(5):  # retry up to ~5 seconds
                try:
                    done_btn = page.query_selector('button:has-text("Done")')
                    if done_btn:
                        done_btn.click()
                        print("    → Clicked Done button")
                        break
                    close_btn = page.query_selector('button:has-text("Close")')
                    if close_btn:
                        close_btn.click()
                        print("    → Clicked Close button")
                        break
                except Exception:
                    pass
                time.sleep(1)
            else:
                print("    → No Done or Close button found after submit")
            print("    → Applied ✓")
            return True

        # --- intermediate steps ---
        elif page.query_selector('button:has-text("Review")'):
            page.click('button:has-text("Review")')

        elif page.query_selector('button:has-text("Next")'):
            page.click('button:has-text("Next")')

        else:
            # Unexpected extra questions — bail
            dismiss_btn = page.query_selector('button[aria-label="Dismiss"]')
            if dismiss_btn:
                dismiss_btn.click()
            print("    → Skipped (extra questions)")
            return False


# ---------------------- SEARCH URL ------------------------

def search_url(page_num: int) -> str:
    base = (
        "https://www.linkedin.com/jobs/search/?"
        f"keywords={KEYWORDS.replace(' ', '%20')}&"
        f"location={LOCATION.replace(' ', '%20')}&"
        "f_AL=true"                # Easy‑Apply filter
    )
    return f"{base}&start={page_num * 25}"


# ------------------------- MAIN ---------------------------

def main():
    applied = skipped = 0

    with sync_playwright() as p:
        browser = p.chromium.launch(headless=HEADLESS, slow_mo=60)
        context = browser.new_context()
        page = context.new_page()

        login(page)

        for pn in range(PAGES):
            url = search_url(pn)
            print(f"\n▶ Scanning page {pn + 1}/{PAGES}: {url}\n")
            page.goto(url, timeout=30000)

            # ---------- robust job‑card lookup (July 2025) ----------
            JOB_SELECTORS = [
                'li[data-occludable-job-id]',          # current markup
                'li.jobs-search-results__list-item',   # older markup
                'div.job-card-container--clickable'    # fallback
            ]

            job_cards = []
            current_selector = None
            for sel in JOB_SELECTORS:
                try:
                    page.wait_for_selector(sel, timeout=7000)
                    job_cards = page.query_selector_all(sel)
                    if job_cards:
                        current_selector = sel
                        break
                except PlaywrightTimeout:
                    continue

            if not job_cards:
                print("   ⚠ No job listings found — skipping this page.")
                continue

            print(f"   Found {len(job_cards)} job cards (selector: {current_selector})")
            # --------------------------------------------------------

            for idx in range(len(job_cards)):
                try:
                    # Re‑obtain handle each time to avoid stale references
                    card = page.query_selector_all(current_selector)[idx]
                    card.scroll_into_view_if_needed()
                    card.click(timeout=3000)
                    page.wait_for_selector('div.jobs-details__main-content', timeout=10000)
                except Exception as exc:
                    print(f"[{idx+1:02}] Could not open job details — {exc} — skipped")
                    skipped += 1
                    continue

                human_sleep(0.8, 1.2)

                if apply_to_job(page):
                    applied += 1
                else:
                    skipped += 1

                human_sleep(1.0, 2.0)

            human_sleep(2.0, 3.0)  # pause between pages

        print(f"\nSummary: Applied to {applied} jobs, skipped {skipped}.")
        browser.close()


if __name__ == "__main__":
    main()
` – main Playwright script  
- `README_usage.md` – setup & run instructions  
- (additional screenshots or logs in `documents/`)

