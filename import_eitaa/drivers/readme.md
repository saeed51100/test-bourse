Yes. Your existing `environment.yml` is fine. Selenium explicitly supports manual driver management when Selenium Manager cannot be used; the current Python API provides `ChromeService(executable_path=...)` for this purpose. ([Selenium][1])

### 1. Keep `environment.yml`

```yaml
channels:
  - conda-forge
  - nodefaults

dependencies:
  - python=3.11
  - pandas
  - openpyxl
  - pip
  - pip:
      - selenium==4.48.0
      - mysql-connector-python
      - tenacity
```

Then:

```bash
conda env create -f environment.yml
conda activate <your-environment>
```

### 2. Check your installed Chrome version

```bash
google-chrome --version
```

For example:

```text
Google Chrome 152.0.7977.82
```

For Chrome 115+, ChromeDriver is distributed through **Chrome for Testing (CfT)**. Chrome's official documentation directs users to the CfT availability dashboard for these versions. ([Chrome for Developers][2])

### 3. Download the matching ChromeDriver

Official Chrome for Testing downloads:

[Chrome for Testing — official availability dashboard](https://googlechromelabs.github.io/chrome-for-testing/?utm_source=chatgpt.com)

For Ubuntu 64-bit, download:

```text
chromedriver-linux64.zip
```

Use the ChromeDriver version corresponding to your installed Chrome version.

For example, if your Chrome is:

```text
152.0.7977.82
```

the corresponding driver URL is:

[ChromeDriver 152.0.7977.82 — Linux 64-bit](https://storage.googleapis.com/chrome-for-testing-public/152.0.7977.82/linux64/chromedriver-linux64.zip?utm_source=chatgpt.com)

The official CfT distribution currently publishes `chromedriver` Linux 64-bit archives alongside the Chrome versions. ([Chrome Labs][3])

### 4. Put the driver in your project

Recommended project structure:

```text
my-project/
├── environment.yml
├── drivers/
│   └── chromedriver
├── notebooks/
│   └── test.ipynb
└── src/
```

Extract the ZIP and place the executable here:

```text
drivers/chromedriver
```

Then:

```bash
chmod +x drivers/chromedriver
```

### 5. Explicitly configure Selenium to use it

In your notebook:

```python
from pathlib import Path

from selenium import webdriver
from selenium.webdriver.chrome.service import Service

PROJECT_ROOT = Path.cwd().parent  # adjust if your notebook is elsewhere
CHROMEDRIVER = PROJECT_ROOT / "drivers" / "chromedriver"

service = Service(executable_path=str(CHROMEDRIVER))

driver = webdriver.Chrome(service=service)

driver.get("https://www.google.com")
```

`Service(executable_path=...)` is the current Selenium-supported mechanism for explicitly specifying the ChromeDriver executable. ([Selenium][4])

### 6. Better project configuration

If the notebook location can vary, use an environment variable rather than hard-coding the path:

```bash
export CHROMEDRIVER_PATH="$PWD/drivers/chromedriver"
```

Then:

```python
import os

from selenium import webdriver
from selenium.webdriver.chrome.service import Service

driver_path = os.environ["CHROMEDRIVER_PATH"]

driver = webdriver.Chrome(
    service=Service(executable_path=driver_path)
)
```

This completely bypasses Selenium Manager's driver-download fallback because you explicitly provide the driver. Selenium's documentation states that Selenium Manager is used when a driver has **not** been provided; manual driver management remains supported. ([Selenium][1])

### 7. Verify

```python
print(driver.capabilities["browserVersion"])
print(driver.capabilities["chrome"]["chromedriverVersion"])
```

Then:

```python
driver.quit()
```

**Recommended setup:** keep `selenium==4.48.0` in `environment.yml`, store the Linux `chromedriver` executable under `drivers/`, pin/document the Chrome + ChromeDriver versions, and pass the executable through `Service(...)`. This is the current Selenium-supported manual-management path. ([Selenium][4])

[1]: https://www.selenium.dev/documentation/selenium_manager/?utm_source=chatgpt.com "Selenium Manager | Selenium"
[2]: https://developer.chrome.com/docs/chromedriver/downloads/?utm_source=chatgpt.com "Downloads  |  ChromeDriver  |  Chrome for Developers"
[3]: https://googlechromelabs.github.io/chrome-for-testing/?channel=00000000000000000000000040000001&clienttype=8&utm_source=chatgpt.com "Chrome for Testing availability"
[4]: https://www.selenium.dev/selenium/docs/api/py/selenium_webdriver_chrome/selenium.webdriver.chrome.service.html?utm_source=chatgpt.com "selenium.webdriver.chrome.service — Selenium 4.48.0 documentation"
