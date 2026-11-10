Yes. For this job I would use **Selenium + Chrome + MariaDB/MySQL**, with the browser session kept persistent and a database row acting as the source of truth for resume/retry state.

One important correction first: **phpMyAdmin is only the web administration interface**. Python should connect directly to the MariaDB/MySQL server that phpMyAdmin manages, using `mysql-connector-python` or SQLAlchemy. You do not automate phpMyAdmin itself.

I also can't reliably give you Eitaa's current CSS selectors from public documentation—the selectors are implementation details of the live web application. Eitaa does have public repositories/documentation, but they don't expose a stable DOM contract for the contacts UI. ([GitHub][1])

If you upload the actual Excel file, I can also verify the column names/types and adapt the import code exactly to it:

## 1. Recommended stack

I'd use:

* **Selenium 4** — Chrome automation
* **webdriver-manager** is optional; modern Selenium can generally manage the driver automatically, so I would initially omit it.
* **pandas + openpyxl** — Excel
* **mysql-connector-python** — MariaDB/MySQL
* Python's built-in `logging` — application logs
* `tenacity` — optional retry/backoff handling

Selenium is a good fit here because you need to interact with a real authenticated browser session and wait for dynamically rendered UI elements. Selenium supports CSS, ID, XPath and other locator strategies; its own guidance recommends stable IDs first, then compact CSS selectors where possible. ([Selenium][2])

### `environment.yml`

```yaml
name: eitaa-import

channels:
  - conda-forge

dependencies:
  - python=3.11
  - pandas
  - openpyxl
  - selenium
  - pip
  - pip:
      - mysql-connector-python
      - tenacity
```

I would **not** install ChromeDriver manually initially. Test:

```bash
conda env create -f environment.yml
conda activate eitaa-import

python -c "import selenium; print(selenium.__version__)"
```

---

# 2. Database design

Use a single `contacts` table.

The important property is that **`phone_normalized` is unique**.

That gives you idempotency: if the program crashes after successfully adding a contact but before recording the result, you can detect the contact on the next run instead of blindly inserting it again.

```sql
CREATE DATABASE IF NOT EXISTS eitaa_import
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE eitaa_import;

CREATE TABLE IF NOT EXISTS contacts (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,

    full_name VARCHAR(500) NOT NULL,
    phone_original VARCHAR(100) NOT NULL,
    phone_normalized VARCHAR(30) NOT NULL,

    status ENUM(
        'pending',
        'processing',
        'success',
        'failed'
    ) NOT NULL DEFAULT 'pending',

    attempts INT NOT NULL DEFAULT 0,

    error_message TEXT NULL,

    started_at DATETIME NULL,
    completed_at DATETIME NULL,

    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL
        DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP,

    PRIMARY KEY (id),
    UNIQUE KEY uq_phone (phone_normalized),
    KEY ix_status_id (status, id)
) ENGINE=InnoDB;
```

I'd also add a separate run log:

```sql
CREATE TABLE IF NOT EXISTS import_runs (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,

    started_at DATETIME NOT NULL,
    finished_at DATETIME NULL,

    total INT NOT NULL DEFAULT 0,
    successful INT NOT NULL DEFAULT 0,
    failed INT NOT NULL DEFAULT 0,

    PRIMARY KEY (id)
) ENGINE=InnoDB;
```

You can inspect everything conveniently through your existing phpMyAdmin installation, but the automation itself talks directly to MariaDB/MySQL.

---

# 3. Normalize Iranian phone numbers

This is particularly important because your Excel file might contain:

```text
9121234567
09121234567
+989121234567
00989121234567
```

but Eitaa's form already supplies `+98`.

I'd normalize all of them internally to:

```text
9121234567
```

and send only those digits to the phone field.

```python
import re


def normalize_iran_phone(value: str) -> str:
    """
    Return Iranian mobile number in local 10-digit form:
        9121234567

    Accepted examples:
        9121234567
        09121234567
        +989121234567
        00989121234567
    """

    if value is None:
        raise ValueError("Phone number is empty")

    s = str(value).strip()

    # Remove spaces, -, parentheses, etc.
    s = re.sub(r"[^\d+]", "", s)

    if s.startswith("+98"):
        s = s[3:]
    elif s.startswith("0098"):
        s = s[4:]
    elif s.startswith("98") and len(s) == 12:
        s = s[2:]
    elif s.startswith("0"):
        s = s[1:]

    if not re.fullmatch(r"9\d{9}", s):
        raise ValueError(f"Invalid Iranian mobile number: {value!r}")

    return s
```

---

# 4. Import Excel into the database

Assuming your Excel columns are exactly:

```text
Full Name
Phone Number
```

I'd import them **before opening Selenium**.

```python
from pathlib import Path

import pandas as pd
import mysql.connector


EXCEL_FILE = Path(
    "/home/saeed/repositories/contacts/import_eitaa.xlsx"
)


def get_db_connection():
    return mysql.connector.connect(
        host="127.0.0.1",
        port=3306,
        user="YOUR_DB_USER",
        password="YOUR_DB_PASSWORD",
        database="eitaa_import",
        charset="utf8mb4",
    )


def import_excel():
    df = pd.read_excel(
        EXCEL_FILE,
        dtype={"Full Name": str, "Phone Number": str},
    )

    expected = {"Full Name", "Phone Number"}

    if set(df.columns) != expected:
        raise ValueError(
            f"Expected columns {expected}, got {set(df.columns)}"
        )

    conn = get_db_connection()

    try:
        cursor = conn.cursor()

        sql = """
            INSERT INTO contacts (
                full_name,
                phone_original,
                phone_normalized,
                status
            )
            VALUES (%s, %s, %s, 'pending')
            ON DUPLICATE KEY UPDATE
                full_name = VALUES(full_name),
                phone_original = VALUES(phone_original)
        """

        inserted = 0

        for _, row in df.iterrows():
            full_name = str(row["Full Name"]).strip()
            phone_original = str(row["Phone Number"]).strip()

            if not full_name:
                continue

            try:
                phone_normalized = normalize_iran_phone(
                    phone_original
                )
            except ValueError as exc:
                print(
                    f"Invalid phone number "
                    f"{phone_original!r}: {exc}"
                )
                continue

            cursor.execute(
                sql,
                (
                    full_name,
                    phone_original,
                    phone_normalized,
                ),
            )

            inserted += 1

        conn.commit()

        print(f"Imported/updated {inserted} contacts.")

    finally:
        cursor.close()
        conn.close()


if __name__ == "__main__":
    import_excel()
```

The `ON DUPLICATE KEY UPDATE` is important.

You can safely run the Excel-import step again without creating 5,000 duplicate database records.

---

# 5. Chrome session persistence

**Do not automate the Eitaa login/OTP process.**

Instead, create a dedicated Chrome profile and log into Eitaa manually once.

For example:

```text
/home/saeed/.config/eitaa-chrome
```

Then Selenium launches Chrome using that profile.

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options


CHROME_PROFILE = "/home/saeed/.config/eitaa-chrome"


def create_driver():
    options = Options()

    options.add_argument(
        f"--user-data-dir={CHROME_PROFILE}"
    )

    options.add_argument("--start-maximized")

    driver = webdriver.Chrome(options=options)

    driver.set_page_load_timeout(60)

    return driver
```

First run:

```python
driver = create_driver()
driver.get("https://web.eitaa.com/")
```

Log into Eitaa manually.

Close the browser normally.

On subsequent executions Selenium will reuse that Chrome profile/session.

This is much more robust than trying to automate OTP authentication.

---

# 6. Eitaa selectors: don't guess them

This is the part I would **not hard-code based on examples from the Internet**.

Open Eitaa manually:

```text
https://web.eitaa.com/
```

Then:

1. Open Chrome DevTools.
2. Press `F12`.
3. Select **Elements**.
4. Navigate to Contacts.
5. Click **Add Contact**.
6. Inspect:

   * Contacts button
   * Add Contact button
   * First-name input
   * Last-name input
   * Phone input
   * Save/Add button

Look for stable attributes such as:

```html
id="..."
name="..."
aria-label="..."
data-testid="..."
```

Prefer those.

For example, if inspection shows:

```html
<input name="first_name">
```

use:

```python
(By.NAME, "first_name")
```

rather than a fragile selector such as:

```python
(By.XPATH, "/html/body/div[2]/div[3]/...")
```

Selenium explicitly supports these locator strategies, and stable IDs/names/CSS selectors are preferable to overly complicated XPath expressions. ([Selenium][3])

---

# 7. Put selectors in one place

Don't scatter selectors throughout your program.

For example:

```python
from selenium.webdriver.common.by import By


class Selectors:

    # Replace these after inspecting Eitaa.

    CONTACTS_BUTTON = (
        By.CSS_SELECTOR,
        "REPLACE_ME"
    )

    ADD_CONTACT_BUTTON = (
        By.CSS_SELECTOR,
        "REPLACE_ME"
    )

    FIRST_NAME = (
        By.CSS_SELECTOR,
        "REPLACE_ME"
    )

    LAST_NAME = (
        By.CSS_SELECTOR,
        "REPLACE_ME"
    )

    PHONE = (
        By.CSS_SELECTOR,
        "REPLACE_ME"
    )

    SAVE_BUTTON = (
        By.CSS_SELECTOR,
        "REPLACE_ME"
    )
```

This makes an Eitaa UI change a five-minute maintenance job rather than requiring you to rewrite the automation.

---

# 8. Reliable waits

Avoid:

```python
time.sleep(3)
```

as the primary synchronization mechanism.

Use `WebDriverWait`.

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


WAIT_SECONDS = 30


def wait_visible(driver, locator):
    return WebDriverWait(
        driver,
        WAIT_SECONDS
    ).until(
        EC.visibility_of_element_located(locator)
    )


def wait_clickable(driver, locator):
    return WebDriverWait(
        driver,
        WAIT_SECONDS
    ).until(
        EC.element_to_be_clickable(locator)
    )
```

And for a form:

```python
first_name = wait_visible(
    driver,
    Selectors.FIRST_NAME
)

last_name = wait_visible(
    driver,
    Selectors.LAST_NAME
)

phone = wait_visible(
    driver,
    Selectors.PHONE
)
```

---

# 9. The actual Eitaa contact operation

The important part is:

```python
first_name = full_name
last_name = ""
phone = phone_normalized
```

Not:

```python
first_name, last_name = full_name.split(...)
```

Your implementation should therefore be:

```python
def add_contact(driver, full_name, phone_normalized):
    """
    Add exactly one Eitaa contact.

    Full name intentionally goes entirely into
    the first-name field.
    """

    # Open contacts.
    wait_clickable(
        driver,
        Selectors.CONTACTS_BUTTON
    ).click()

    # Open Add Contact.
    wait_clickable(
        driver,
        Selectors.ADD_CONTACT_BUTTON
    ).click()

    # First name = COMPLETE full name.
    first_name = wait_visible(
        driver,
        Selectors.FIRST_NAME
    )

    first_name.clear()
    first_name.send_keys(full_name)

    # Last name must remain completely blank.
    last_name = wait_visible(
        driver,
        Selectors.LAST_NAME
    )

    last_name.clear()

    # Eitaa already displays +98.
    phone = wait_visible(
        driver,
        Selectors.PHONE
    )

    phone.clear()
    phone.send_keys(phone_normalized)

    # Submit.
    wait_clickable(
        driver,
        Selectors.SAVE_BUTTON
    ).click()

    # IMPORTANT:
    # Wait for confirmation/state change here.
    #
    # Replace this with a selector for whatever
    # Eitaa actually displays after successful
    # contact creation.
```

The last part is deliberately not invented.

You need to inspect what Eitaa does after clicking Save:

* Does the dialog disappear?
* Does a toast appear?
* Does the contact appear in the list?
* Does a success message appear?
* Does the form remain open?

Then create a success condition based on that state.

For example, **if** the dialog disappears:

```python
WebDriverWait(driver, 30).until(
    EC.invisibility_of_element_located(
        Selectors.FIRST_NAME
    )
)
```

Or **if** a success notification appears:

```python
WebDriverWait(driver, 30).until(
    EC.visibility_of_element_located(
        Selectors.SUCCESS_MESSAGE
    )
)
```

That confirmation is critical.

You don't want:

```text
click Save
→ immediately mark database row successful
```

because the browser may have clicked the button but Eitaa may have rejected the operation.

---

# 10. Database state machine

I'd use:

```text
pending
   │
   ▼
processing
   │
   ├── success
   │
   └── failed
```

When starting an operation:

```sql
UPDATE contacts
SET
    status = 'processing',
    attempts = attempts + 1,
    started_at = NOW()
WHERE id = %s;
```

Success:

```sql
UPDATE contacts
SET
    status = 'success',
    error_message = NULL,
    completed_at = NOW()
WHERE id = %s;
```

Failure:

```sql
UPDATE contacts
SET
    status = 'failed',
    error_message = %s
WHERE id = %s;
```

---

# 11. Reset interrupted contacts

Suppose Ubuntu crashes while a contact is `processing`.

On startup:

```sql
UPDATE contacts
SET status = 'pending'
WHERE status = 'processing';
```

Or, better, only reset stale operations:

```sql
UPDATE contacts
SET status = 'pending'
WHERE status = 'processing'
  AND started_at < NOW() - INTERVAL 30 MINUTE;
```

This lets the next execution recover interrupted work.

---

# 12. Fetch the next contact

```python
def get_next_contact(conn):
    cursor = conn.cursor(dictionary=True)

    try:
        cursor.execute("""
            SELECT
                id,
                full_name,
                phone_original,
                phone_normalized,
                attempts
            FROM contacts
            WHERE status IN ('pending', 'failed')
              AND attempts < 3
            ORDER BY id
            LIMIT 1
        """)

        return cursor.fetchone()

    finally:
        cursor.close()
```

You may want failed contacts handled separately rather than automatically retried. For 5,000 records, I generally recommend:

```text
pending → automatic retry
failed  → max 2-3 attempts
```

and then review the failures manually.

---

# 13. Complete processing loop

Here is the basic architecture I would use:

```python
import logging
from datetime import datetime

from selenium.common.exceptions import (
    TimeoutException,
    WebDriverException,
)


logging.basicConfig(
    filename="eitaa_import.log",
    level=logging.INFO,
    format=(
        "%(asctime)s | %(levelname)s | "
        "%(message)s"
    ),
)


def mark_processing(conn, contact_id):
    cursor = conn.cursor()

    try:
        cursor.execute("""
            UPDATE contacts
            SET
                status = 'processing',
                attempts = attempts + 1,
                started_at = NOW()
            WHERE id = %s
        """, (contact_id,))

        conn.commit()

    finally:
        cursor.close()


def mark_success(conn, contact_id):
    cursor = conn.cursor()

    try:
        cursor.execute("""
            UPDATE contacts
            SET
                status = 'success',
                error_message = NULL,
                completed_at = NOW()
            WHERE id = %s
        """, (contact_id,))

        conn.commit()

    finally:
        cursor.close()


def mark_failed(conn, contact_id, error):
    cursor = conn.cursor()

    try:
        cursor.execute("""
            UPDATE contacts
            SET
                status = 'failed',
                error_message = %s
            WHERE id = %s
        """, (
            str(error)[:60000],
            contact_id,
        ))

        conn.commit()

    finally:
        cursor.close()


def reset_stale_processing(conn):
    cursor = conn.cursor()

    try:
        cursor.execute("""
            UPDATE contacts
            SET status = 'pending'
            WHERE status = 'processing'
              AND started_at <
                  NOW() - INTERVAL 30 MINUTE
        """)

        conn.commit()

    finally:
        cursor.close()


def run_import():
    conn = get_db_connection()

    driver = None

    try:
        reset_stale_processing(conn)

        driver = create_driver()

        driver.get("https://web.eitaa.com/")

        # At this point, verify that the existing
        # Eitaa session is still authenticated.
        #
        # If not, stop and ask for manual login
        # rather than attempting OTP automatically.

        while True:

            contact = get_next_contact(conn)

            if contact is None:
                logging.info("No more contacts.")
                break

            contact_id = contact["id"]
            full_name = contact["full_name"]
            phone = contact["phone_normalized"]

            logging.info(
                "Processing id=%s name=%r phone=%s",
                contact_id,
                full_name,
                phone,
            )

            mark_processing(conn, contact_id)

            try:

                add_contact(
                    driver,
                    full_name,
                    phone,
                )

                mark_success(
                    conn,
                    contact_id,
                )

                logging.info(
                    "SUCCESS id=%s",
                    contact_id,
                )

            except TimeoutException as exc:

                mark_failed(
                    conn,
                    contact_id,
                    f"Timeout: {exc}",
                )

                logging.exception(
                    "TIMEOUT id=%s",
                    contact_id,
                )

            except WebDriverException as exc:

                mark_failed(
                    conn,
                    contact_id,
                    f"WebDriver error: {exc}",
                )

                logging.exception(
                    "WEBDRIVER ERROR id=%s",
                    contact_id,
                )

                # Browser may be unusable.
                # Recreate it.

                try:
                    driver.quit()
                except Exception:
                    pass

                driver = create_driver()
                driver.get(
                    "https://web.eitaa.com/"
                )

            except Exception as exc:

                mark_failed(
                    conn,
                    contact_id,
                    repr(exc),
                )

                logging.exception(
                    "ERROR id=%s",
                    contact_id,
                )

    finally:

        if driver is not None:
            try:
                driver.quit()
            except Exception:
                pass

        conn.close()
```

---

# 14. One improvement I'd strongly recommend

For a job of **5,000 contacts**, don't treat a failed contact as a fatal error.

Your loop should behave like:

```text
contact 1 → success
contact 2 → success
contact 3 → error → database + log
contact 4 → success
contact 5 → success
...
```

not:

```text
contact 3 → error → entire program exits
```

That's the main reason for having the database state.

---

# 15. Duplicate prevention

There are actually **two different duplicate problems**.

### Duplicate in Excel

The database constraint handles this:

```sql
UNIQUE(phone_normalized)
```

### Contact already exists in Eitaa

This is harder.

Suppose:

```text
Excel:
John Smith    9121234567
```

is already in Eitaa before your automation begins.

Your database doesn't know that.

So before committing to the full 5,000-contact run, test whether Eitaa's Contacts UI lets you search existing contacts.

If it does, you can implement:

```python
if eitaa_contact_exists(driver, phone):
    mark_success(...)
    continue
```

or better:

```text
status = skipped_existing
```

I'd therefore change the ENUM to:

```sql
status ENUM(
    'pending',
    'processing',
    'success',
    'skipped_existing',
    'failed'
)
```

This gives you a much better audit trail.

---

# 16. Don't use "already added" solely based on the database

There is a subtle crash scenario:

```text
1. Click Add
2. Eitaa successfully creates contact
3. Chrome crashes
4. Python never executes mark_success()
```

Database says:

```text
pending
```

but Eitaa already contains the contact.

On resume, your script could create a duplicate.

Therefore, for a **really robust implementation**, the sequence should eventually become:

```text
database says pending
        ↓
check whether phone already exists in Eitaa
        ↓
YES ─────────────→ skipped_existing
        │
        NO
        ↓
add contact
        ↓
verify success
        ↓
success
```

This is much safer than assuming the browser click succeeded.

---

# 17. Chrome crash recovery

Wrap browser-level failures separately from contact-level failures.

For example:

```python
except WebDriverException:

    logging.exception(
        "Chrome/WebDriver failure"
    )

    try:
        driver.quit()
    except Exception:
        pass

    driver = create_driver()

    driver.get(
        "https://web.eitaa.com/"
    )

    # Wait for authenticated Eitaa state again.
```

But don't immediately retry the contact blindly.

First check:

```text
Does contact already exist?
```

because the browser could have crashed **after** Eitaa accepted the request.

---

# 18. Add screenshots for failures

This is extremely useful when automating a UI that can change.

```python
from pathlib import Path
from datetime import datetime


SCREENSHOT_DIR = Path("screenshots")
SCREENSHOT_DIR.mkdir(exist_ok=True)


def save_failure_screenshot(driver, contact_id):
    filename = (
        SCREENSHOT_DIR
        / f"{contact_id}_{datetime.now():%Y%m%d_%H%M%S}.png"
    )

    driver.save_screenshot(str(filename))

    return filename
```

Then:

```python
except Exception as exc:

    screenshot = save_failure_screenshot(
        driver,
        contact_id,
    )

    mark_failed(
        conn,
        contact_id,
        f"{exc}\nScreenshot: {screenshot}",
    )
```

For 5,000 records, this will make debugging dramatically easier.

---

# 19. Logging

Use both:

### Database

Structured information:

```text
id
full_name
phone
status
attempts
error_message
timestamps
```

### Log file

Operational information:

```text
2026-09-01 20:41:12 | INFO | Processing id=152
2026-09-01 20:41:14 | INFO | SUCCESS id=152

2026-09-01 20:41:17 | INFO | Processing id=153
2026-09-01 20:41:48 | ERROR | TIMEOUT id=153
```

That separation is useful.

---

# 20. Recommended project structure

I'd make the project something like:

```text
eitaa-import/
│
├── environment.yml
├── config.py
├── database.py
├── excel_import.py
├── eitaa.py
├── importer.py
├── main.py
│
├── logs/
│   └── eitaa_import.log
│
├── screenshots/
│
└── notebooks/
    └── development.ipynb
```

Even if you prototype in Jupyter, I'd eventually move the actual 5,000-record run into a normal Python script.

Jupyter is excellent for developing and testing the selectors, but a long-running unattended browser automation job is considerably easier to restart and monitor as:

```bash
python main.py
```

---

# 21. Development workflow I recommend

**Do not start with 5,000.**

Use:

```text
1 contact
↓
5 contacts
↓
20 contacts
↓
100 contacts
↓
full import
```

First verify:

### Test 1 — name

Input:

```text
John Michael Smith
```

Eitaa:

```text
First name: John Michael Smith
Last name:
```

### Test 2 — phone

Input:

```text
9121234567
```

Eitaa:

```text
+98 | 9121234567
```

### Test 3 — failure

Temporarily give it an invalid phone number.

Confirm:

```text
status = failed
error_message = ...
```

and that the next contact still runs.

### Test 4 — crash

Kill Chrome during processing.

Restart Python.

Confirm the stale `processing` record returns to `pending`.

### Test 5 — duplicate

Run the same five contacts twice.

The second run should **not blindly create another five contacts**.

---

# 22. One alternative worth knowing about

While Selenium is what I'd recommend for your stated requirement, there is now an unofficial Python Eitaa client, `eitaa-cli`, which claims support for contact management and provides a Python API. Its documentation describes contact management and a Python client, although it is not an official Eitaa library. ([GitHub][4])

So before spending a lot of time reverse-engineering the UI, I'd investigate whether its contact-management API can perform exactly the operation you need.

However, because you specifically want to reproduce the user workflow in **Eitaa Web**, Selenium remains the safer implementation choice.

---

## Recommended final architecture

```text
                     import_eitaa.xlsx
                            │
                            ▼
                     pandas/openpyxl
                            │
                            ▼
                  MariaDB/MySQL database
                            │
                 ┌──────────┴──────────┐
                 │                     │
             pending                failed
                 │
                 ▼
          Selenium + Chrome
                 │
                 ▼
          web.eitaa.com
                 │
          ┌──────┴──────┐
          │             │
       exists          new
          │             │
          ▼             ▼
      skipped        Add Contact
                        │
                        ▼
                 First name =
                 COMPLETE Full Name

                 Last name =
                 EMPTY

                 Phone =
                 9121234567
                        │
                        ▼
                 verify success
                   │          │
                   ▼          ▼
                success     failed
```

The **one piece I would not try to guess** is the current Eitaa DOM selectors and success state. Inspect those in Chrome and put them into `Selectors`; Selenium itself recommends stable IDs/names/compact selectors rather than brittle DOM paths. ([Selenium][5])

Once you have the selectors for the **Contacts**, **Add Contact**, **First name**, **Last name**, **Phone**, **Save**, and **success/duplicate** elements, the rest of the implementation above can remain essentially unchanged.

[1]: https://github.com/orgs/eitaa/repositories?utm_source=chatgpt.com "eitaa repositories · GitHub"
[2]: https://www.selenium.dev/documentation/webdriver/elements/locators/?utm_source=chatgpt.com "Locator strategies | Selenium"
[3]: https://www.selenium.dev/documentation/webdriver/elements/finders/?utm_source=chatgpt.com "Finding web elements | Selenium"
[4]: https://github.com/EhsanAhmadzadeh/eitaa-cli?utm_source=chatgpt.com "GitHub - EhsanAhmadzadeh/eitaa-cli · GitHub"
[5]: https://www.selenium.dev/documentation/test_practices/encouraged/locators/?utm_source=chatgpt.com "Tips on working with locators | Selenium"
