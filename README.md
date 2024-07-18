# GoFundMe Campaign Scraper

This repository contains a Python script that scrapes GoFundMe campaign data for campaigns related to Gaza. The script uses Selenium to automate web browsing, BeautifulSoup for parsing HTML, and pandas for data manipulation and storage.

## Features

- Automatically installs necessary packages if they are not already installed.
- Uses Selenium to load all campaigns by clicking the "Show more" button until all campaigns are loaded.
- Extracts campaign details such as title, organizer name, amount raised, goal, number of donations, and description.
- Saves progress periodically to avoid data loss.
- Saves the final scraped data to a CSV file.

## Prerequisites

- Python 3.x
- Microsoft Edge browser and Edge WebDriver

## Installation

1. **Clone the repository:**
    ```bash
    git clone https://github.com/yourusername/gofundme-scraper.git
    cd gofundme-scraper
    ```

2. **Install necessary packages:**
    The script will automatically install any missing packages, but you can also manually install them using:
    ```bash
    pip install selenium beautifulsoup4 pandas
    ```

3. **Download Edge WebDriver:**
    Download the Edge WebDriver from the [official site](https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/) and place the executable in an accessible location. Update the `edge_driver_path` in the script accordingly.

## Usage

1. **Set the path to the Edge WebDriver executable:**
    ```python
    edge_driver_path = r'C:\path\to\msedgedriver.exe'
    ```

2. **Run the script:**
    ```bash
    python gofundme_scraper.py
    ```

3. **Progress saving:**
    The script saves progress periodically to a JSON file located at `C:\Users\USER\Documents\progress_gofundme2.json`. If the script is interrupted, it will resume from the last saved campaign.

4. **Output:**
    The final data is saved to a CSV file at `C:\Users\USER\Documents\gofundme_campaigns11.csv`.

## Script Details

### Function to Install a Package if It's Not Already Installed

The function `install_and_import` checks if a package is installed. If the package is not found, it installs it using `pip`.

```python
import subprocess
import sys

def install_and_import(package):
    try:
        __import__(package)
    except ImportError:
        print(f"{package} not found. Installing...")
        subprocess.check_call([sys.executable, "-m", "pip", "install", package])
    finally:
        globals()[package] = __import__(package)
```

### Package Installation and Import

The script defines a list of required packages and installs them if necessary. It also imports standard libraries directly.

```python
packages = [
    "selenium",
    "bs4",
    "pandas",
]

for package in packages:
    install_and_import(package)

import time
import json
import os
from selenium import webdriver
from selenium.webdriver.edge.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from bs4 import BeautifulSoup
import pandas as pd

print("All required libraries are installed and imported.")
```

### WebDriver Initialization

The script sets up the Edge WebDriver service and initializes the WebDriver with a retry mechanism.

```python
edge_driver_path = r'C:\Users\USER\Downloads\edgedriver_win64\msedgedriver.exe'
service = Service(edge_driver_path)

def init_driver():
    options = webdriver.EdgeOptions()
    options.add_argument('headless')
    driver = webdriver.Edge(service=service, options=options)
    return driver

driver = init_driver()
```

### Loading GoFundMe Campaigns

The script opens the GoFundMe search page for Gaza and clicks the "Show more" button until all campaigns are loaded.

```python
url = 'https://www.gofundme.com/s?q=gaza'
driver.get(url)

while True:
    try:
        show_more_button = WebDriverWait(driver, 10).until(
            EC.element_to_be_clickable((By.XPATH, '//button[@data-element-id="btn_show_more"]'))
        )
        show_more_button.click()
        time.sleep(5)
    except:
        break

page_source = driver.page_source
soup = BeautifulSoup(page_source, 'html.parser')
```

### Extracting Campaign Data

The script extracts campaign links, titles, and organizer names.

```python
campaign_divs = soup.find_all('a', class_='full-state-list-card_actionCard__c6v8R hrt-action-card hrt-base-button')
campaign_links = ['https://www.gofundme.com' + a['href'] for a in campaign_divs]

campaign_data = []
divs_titles = soup.find_all('div', class_='full-state-list-card_truncate__79X_9 full-state-list-card_fundName__oq_nD')
divs_organizers = soup.find_all('div', class_='full-state-list-card_organizer__j97E8 full-state-list-card_truncateSingle__AtDr_ hrt-text-body-sm hrt-text-gray hrt-font-regular')

for i in range(len(campaign_links)):
    title = divs_titles[i].get_text() if i < len(divs_titles) else ""
    organizer = divs_organizers[i].find('span').get_text().strip() if i < len(divs_organizers) else ""
    campaign_data.append({
        'Campaign Link': campaign_links[i],
        'Campaign Title': title,
        'Organizer Name': organizer,
        'Amount Raised': "",
        'Goal': "",
        'Number of Donations': "",
        'Description': ""
    })
```

### Saving Progress

The script saves progress to a JSON file and loads it if it exists to resume scraping from where it left off.

```python
progress_file_path = r'C:\Users\USER\Documents\progress_gofundme2.json'

if os.path.exists(progress_file_path):
    try:
        with open(progress_file_path, 'r') as file:
            progress = json.load(file)
            campaign_data = progress.get('campaign_data', campaign_data)
            start_index = progress.get('last_index', -1) + 1
            print(f"Loaded progress: start_index={start_index}")
    except (json.JSONDecodeError, KeyError) as e:
        print(f"Error loading progress file: {e}")
        start_index = 0
else:
    start_index = 0
```

### Scraping Campaign Details

The script iterates over each campaign, opens the campaign page, and extracts data such as amount raised, goal, number of donations, and description.

```python
for index in range(start_index, len(campaign_data)):
    link = campaign_data[index]['Campaign Link']
    try:
        driver.get(link)

        WebDriverWait(driver, 20).until(
            EC.presence_of_element_located((By.CLASS_NAME, 'hrt-disp-inline'))
        )

        retry_count = 4
        for attempt in range(retry_count):
            try:
                read_more_button = WebDriverWait(driver, 10).until(
                    EC.element_to_be_clickable((By.CSS_SELECTOR, 'button[data-element-id="btn_story_read-more"]'))
                )
                read_more_button.click()
                WebDriverWait(driver, 10).until(
                    EC.staleness_of(read_more_button)
                )
                break
            except Exception as e:
                if attempt < retry_count - 1:
                    print(f"Retrying ({attempt+1}/{retry_count})...")
                    time.sleep(5)
                else:
                    print(f"Read more button not found or not clickable: {e}")

        campaign_page_source = driver.page_source
        campaign_soup = BeautifulSoup(campaign_page_source, 'html.parser')

        amount_raised_text = campaign_soup.find('div', class_='hrt-disp-inline').get_text().strip() if campaign_soup.find('div', class_='hrt-disp-inline') else ""
        goal_text = campaign_soup.find('span', class_='hrt-text-body-sm hrt-text-gray').get_text().strip() if campaign_soup.find('span', class_='hrt-text-body-sm hrt-text-gray') else ""
        donations_no_text = campaign_soup.find('span', class_='hrt-text-gray-dark').get_text().strip() if campaign_soup.find('span', class_='hrt-text-gray-dark') else ""

        campaign_data[index]['Amount Raised'] = amount_raised_text
        campaign_data[index]['Goal'] = goal_text
        campaign_data[index]['Number of Donations'] = donations_no_text

        description_div = campaign_soup.find('div', class_='campaign-description_content__C1C_5 hrt-mt-3 campaign-description_isOpen__4cQeG')
        description_text = ''
        if description_div:
            for child in description_div.find_all(recursive=False):
                if child.name == 'div' and 'img' not in child.attrs:
                    description_text += child.get_text(separator=' ', strip=True) + ' '

        campaign_data[index]['Description'] = description_text.strip()

        if index % 10 == 0:
            with open(progress_file_path, 'w') as file:
                json.dump({
                    'campaign_data': campaign_data,
                    'last_index': index
                }, file)
            print(f"Progress saved at index {index}")

    except Exception as e:
        print(f"An error occurred: {e}")
        driver.quit()
        driver = init_driver()
        continue



with open(progress_file_path, 'w') as file:
    json.dump({
        'campaign_data': campaign_data,
        'last_index': len(campaign_data) - 1
    }, file)

driver.quit()
```

### Saving Data to CSV

The script converts the list of dictionaries to a pandas DataFrame and saves it to a CSV file.

```python
df = pd.DataFrame(campaign_data)
df.to_csv(r'C:\Users\USER\Documents\gofundme_campaigns11.csv', index=False)
print("Data saved to CSV.")
```

Contributing
Contributions are welcome! Please fork the repository and create a pull request with your changes.
