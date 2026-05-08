import time
import re
import os
import datetime
import threading
import urllib.parse
from queue import Queue
from urllib.parse import urlparse
import requests
from io import BytesIO
import openpyxl
import pytesseract
from PIL import Image
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from threading import Lock
EMAIL_REGEX = r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b"
FILTER_STRINGS = [
    "@3x.dd058ad.webp", "@sentry.wixpress.com", "@sentry.io", "@webador.com", "@o160250.ingest.sentry.io", "@company.com", "@domain.com", "@yourname.com", "@1.2x.jpg",
    "@ndiscovered.com", "@pixelspread.com", "@indiantypefoundry.com", "@2x.jpeg", "@o4506225101766656.ingest.sentry.io","@o46801.ingest.us.sentry.io"
]
IMAGE_EXTENSIONS = (".png", ".jpg", ".jpeg", ".webp", ".gif", ".svg", ".js")
def filter_emails(email_list, filter_list=FILTER_STRINGS):
    filtered = []
    for email in email_list:
        email_lower = email.lower()
        if any(sub in email for sub in filter_list):
            continue
        if email_lower.endswith(IMAGE_EXTENSIONS):
            continue
        filtered.append(email)
    return filtered
def extract_text_from_image(image):
    return pytesseract.image_to_string(image)
def scrape_emails_from_page_source(page_source):
    emails = re.findall(EMAIL_REGEX, page_source, re.IGNORECASE)
    try:
        decoded_source = urllib.parse.unquote(page_source)
        decoded_emails = re.findall(EMAIL_REGEX, decoded_source, re.IGNORECASE)
        emails.extend(decoded_emails)
    except Exception:
        pass
    return list(set(emails))
def scroll_down_page(driver):
    last_height = driver.execute_script("return document.body.scrollHeight")
    while True:
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
        time.sleep(2)
        new_height = driver.execute_script("return document.body.scrollHeight")
        if new_height == last_height:
            break
        last_height = new_height
def extract_emails_from_image(image_url):
    try:
        response = requests.get(image_url)
        img = Image.open(BytesIO(response.content))
        text = extract_text_from_image(img)
        return re.findall(EMAIL_REGEX, text, re.IGNORECASE)
    except Exception as e:
        return []
def is_internal_link(link, base_domain):
    parsed_link = urlparse(link)
    return parsed_link.netloc == base_domain
def extract_links_with_selenium(driver, base_url):
    try:
        driver.get(base_url)
        WebDriverWait(driver, 20).until(
            EC.presence_of_all_elements_located((By.TAG_NAME, "a"))
        )
        elements = driver.find_elements(By.TAG_NAME, "a")
        base_domain = urlparse(base_url).netloc
        internal_links = []
        facebook_links = []
        linktree_links = []
        for element in elements:
            href = element.get_attribute('href')
            if not href:
                continue
            if 'facebook.com' in href:
                facebook_links.append(href)
            elif 'linktr.ee' in href:
                linktree_links.append(href)
            elif is_internal_link(href, base_domain):
                internal_links.append(href)
        return internal_links, facebook_links, linktree_links
    except Exception as e:
        return [], [], []
def extract_emails_with_waits_and_decoding(driver):
    emails = set()
    try:
        WebDriverWait(driver, 15).until(
            EC.presence_of_element_located((By.TAG_NAME, "body"))
        )
    except:
        pass
    driver.execute_script("window.scrollTo(0, document.body.scrollHeight/2);")
    time.sleep(3)
    driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
    time.sleep(3)
    page_source = driver.page_source
    emails.update(scrape_emails_from_page_source(page_source))
    try:
        elements = driver.find_elements(By.XPATH, "//*[@href]")
        for el in elements:
            href = el.get_attribute("href")
            if href:
                decoded_href = urllib.parse.unquote(href)
                found = re.findall(EMAIL_REGEX, decoded_href, re.IGNORECASE)
                emails.update(found)
    except Exception:
        pass
    return list(emails)
def extract_linktree_specific_emails(driver):
    emails = set()
    try:
        elements = driver.find_elements(By.XPATH, "//a[starts-with(@href, 'mailto:')]")
        for el in elements:
            href = el.get_attribute("href")
            if href:
                email = href.replace("mailto:", "").split("?")[0].strip()
                if re.match(EMAIL_REGEX, email, re.IGNORECASE):
                    emails.add(email)
    except Exception:
        pass
    try:
        elements = driver.find_elements(By.XPATH, "//p | //span | //h1 | //h2 | //a")
        for el in elements:
            text = el.text
            if text and "@" in text:
                found = re.findall(EMAIL_REGEX, text, re.IGNORECASE)
                emails.update(found)
    except Exception:
        pass
    try:
        scripts = driver.find_elements(By.TAG_NAME, "script")
        for script in scripts:
            script_content = script.get_attribute("innerHTML")
            if script_content and "@" in script_content:
                decoded_script = urllib.parse.unquote(script_content)
                found = re.findall(EMAIL_REGEX, decoded_script, re.IGNORECASE)
                emails.update(found)
    except Exception:
        pass
    return list(emails)
def process_website(driver, url):
    print(f"\nProcessing links for {url}")
    internal_links, facebook_links, linktree_links = extract_links_with_selenium(driver, url)
    internal_links_visited = 0
    facebook_links_visited = 0
    linktree_links_visited = 0
    MAX_INTERNAL_LINKS = 5
    MAX_FACEBOOK_LINKS = 10
    MAX_LINKTREE_LINKS = 10
    internal_emails = set()
    facebook_emails = set()
    linktree_emails = set()
    for link in internal_links:
        if internal_links_visited >= MAX_INTERNAL_LINKS:
            break
        try:
            driver.get(link)
            internal_emails.update(extract_emails_with_waits_and_decoding(driver))
            internal_links_visited += 1
        except Exception:
            pass
    for link in facebook_links:
        if facebook_links_visited >= MAX_FACEBOOK_LINKS:
            break
        try:
            driver.get(link)
            facebook_emails.update(extract_emails_with_waits_and_decoding(driver))
            facebook_links_visited += 1
        except Exception:
            pass
    for link in linktree_links:
        if linktree_links_visited >= MAX_LINKTREE_LINKS:
            break
        try:
            driver.get(link)
            linktree_emails.update(extract_emails_with_waits_and_decoding(driver))
            linktree_emails.update(extract_linktree_specific_emails(driver))
            linktree_links_visited += 1
            print(f"Processed Linktree link: {link} - Found emails: {linktree_emails}")
        except Exception as e:
            print(f"Error processing Linktree: {link} - Error: {e}")
    return list(internal_emails), list(facebook_emails), list(linktree_emails)
def setup_driver():
    chrome_options = webdriver.ChromeOptions()
    chrome_options.add_argument("--disable-gpu")
    chrome_options.add_argument("--no-sandbox")
    chrome_options.add_argument("--disable-popup-blocking")
    chrome_options.add_argument("--disable-notifications")
    chrome_options.add_argument("--disable-extensions")
    chrome_options.add_argument("--disable-save-password-bubble")
    chrome_options.add_argument("--safebrowsing-disable-download-protection")
    chrome_options.add_argument("--safebrowsing-disable-extension-blacklist")
    prefs = {
        "download.prompt_for_download": False,
        "download.default_directory": os.devnull,
        "download_restrictions": 3,
        "profile.default_content_setting_values.automatic_downloads": 2,
        "profile.default_content_setting_values.popups": 2
    }
    chrome_options.add_experimental_option("prefs", prefs)
    chrome_options.add_argument(
        "user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36")
    driver = webdriver.Chrome(
        service=Service(ChromeDriverManager().install()),
        options=chrome_options
    )
    return driver
website_email_dict = {}
dict_lock = Lock()
processed_count = 0
total_urls = 0
def process_url_queue(url_queue, thread_id):
    global processed_count, total_urls
    driver = setup_driver()

    while not url_queue.empty():
        url = url_queue.get()
        try:
            print(f"\nThread {thread_id}: Processing URL: {url}")

            with dict_lock:
                website_email_dict[url] = {
                    "direct": [],
                    "selenium_internal": [],
                    "search_engine": [],
                    "facebook": [],
                    "linktree": []
                }
            driver.get(url)
            time.sleep(2)
            page_source = driver.page_source
            direct_emails_visible = scrape_emails_from_page_source(page_source)
            scroll_down_page(driver)
            page_source = driver.page_source
            direct_emails_hidden = scrape_emails_from_page_source(page_source)
            direct_emails = list(set(direct_emails_visible + direct_emails_hidden))
            with dict_lock:
                website_email_dict[url]["direct"].extend(direct_emails)
            selenium_internal_emails, facebook_emails, linktree_emails = process_website(driver, url)
            with dict_lock:
                website_email_dict[url]["selenium_internal"].extend(selenium_internal_emails)
                website_email_dict[url]["facebook"].extend(facebook_emails)
                website_email_dict[url]["linktree"].extend(linktree_emails)
                query = f"{url} email".replace(' ', '+')
                search_engines = {
                    "Yahoo": "https://search.yahoo.com/search?p=",
                    "Bing": "https://www.bing.com/search?q=",
                    "DuckDuckGo": "https://duckduckgo.com/?q=",
                    "AOL": "https://search.aol.com/aol/search?q=",
                    "Ecosia": "https://www.ecosia.org/search?q=",
                    "Excite": "https://results.excite.com/serp?q=",
                }
            for engine_name, engine_url in search_engines.items():
                driver.get(f"{engine_url}{query}")
                time.sleep(2)
                search_engine_emails = scrape_emails_from_page_source(driver.page_source)
                with dict_lock:
                    website_email_dict[url]["search_engine"].extend(search_engine_emails)
        except Exception as e:
            print(f"Thread {thread_id}: Error with {url}: {e}")
        finally:
            with dict_lock:
                processed_count += 1
                pending_count = total_urls - processed_count
                print(f"\n--- [UPDATE] Processed: {processed_count}/{total_urls} | Pending: {pending_count} ---")
            url_queue.task_done()
    driver.quit()


urls_string = """
https://www.wright.edu/dining-services
https://www.funfitnesswithtasha.com/
http://monarchmartialarts.com
https://www.yuppentertainment.com/contact
https://blackwomeninmotion.org/stay-in-touch
http://www.vibegymandwellness.com
https://www.alex-goulding.com/
https://www.ubcdanceclub.com/
http://www.thedancerbody.com
"""
urls = [url.strip() for url in urls_string.strip().split('\n') if url.strip()]
total_urls = len(urls)
url_queue = Queue()
for url in urls:
    url_queue.put(url)
print(f"Starting execution for {total_urls} websites...")
start_time = time.time()
num_threads = 10
threads = []
for i in range(num_threads):
    thread = threading.Thread(target=process_url_queue, args=(url_queue, i + 1))
    thread.start()
    threads.append(thread)
for thread in threads:
    thread.join()
remove_emails = [
    "typesetit@att.net", "abuse@bodis.com", "example@mysite.com", "hello@businessname.com", "sales@youcanbook.me", "yourdomain@mail.com", "username@gmail.com",
    "myemail@mydomain.com", "kamran.a@wellnessliving.com", "mmccormick@contoso.com", "y@gmail.com", "example@domain.com", "you@address.com", "contact@mysite.com", "name@yoursite.com"
]
remove_emails_set = set(email.lower() for email in remove_emails)
wb = openpyxl.Workbook()
ws = wb.active
ws.append(["Website", "Direct Emails", "Internal Page Emails",
           "Search Engine Emails", "Facebook Emails", "Linktree Emails"])
for website, data in website_email_dict.items():
    direct_emails_all = set(filter_emails(data["direct"]))
    selenium_internal_emails_all = set(filter_emails(data["selenium_internal"]))
    search_engine_emails_all = set(filter_emails(data["search_engine"]))
    facebook_emails_all = set(filter_emails(data["facebook"]))
    linktree_emails_all = set(filter_emails(data["linktree"]))
    direct_emails = {email for email in direct_emails_all if email.lower() not in remove_emails_set}
    selenium_internal_emails = {email for email in selenium_internal_emails_all if
                                email.lower() not in remove_emails_set}
    search_engine_emails = {email for email in search_engine_emails_all if email.lower() not in remove_emails_set}
    facebook_emails = {email for email in facebook_emails_all if email.lower() not in remove_emails_set}
    linktree_emails = {email for email in linktree_emails_all if email.lower() not in remove_emails_set}
    selenium_internal_emails -= direct_emails
    search_engine_emails -= (direct_emails | selenium_internal_emails)
    facebook_emails -= (direct_emails | selenium_internal_emails | search_engine_emails)
    row = [
        website,
        ', '.join(sorted(direct_emails)),
        ', '.join(sorted(selenium_internal_emails)),
        ', '.join(sorted(search_engine_emails)),
        ', '.join(sorted(facebook_emails)),
        ', '.join(sorted(linktree_emails))
    ]
    ws.append(row)
output_dir = r"C:\Users\KamranAhmed\Downloads"
timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
filename = f"USnotfoundweekenddata_{timestamp}.xlsx"
output_path = os.path.join(output_dir, filename)
if not os.path.exists(output_dir):
    os.makedirs(output_dir)
wb.save(output_path)
print(f"\nReport generated successfully at: {output_path}")
keywords_to_filter = ["John.Smith", "JohnSmith", "jane.doe", "JSmith", "jdoe", "john", "First.Last", "FirstLast",
                      "xxxx", "janedoe", "first", "Jane", "webmaster", "flast", "last"]
for row in range(2, ws.max_row + 1):
    for col in range(2, 7):
        cell = ws.cell(row=row, column=col)
        if cell.value:
            email_list = [email.strip() for email in cell.value.split(',')]
            filtered_emails = [
                email for email in email_list
                if not any(email.lower().startswith(keyword.lower())
                           for keyword in keywords_to_filter)
            ]
            cell.value = ", ".join(filtered_emails)
wb.save(output_path)
print(f"Fully filtered report (Columns B–F) saved successfully at: {output_path}")
end_time = time.time()
total_time = end_time - start_time
minutes, seconds = divmod(total_time, 60)
print("\n========================================================")
print("                    EXECUTION SUMMARY                   ")
print("========================================================")
print(f"Total Websites Provided : {total_urls}")
print(f"Total Websites Processed: {processed_count}")
print(f"Total Time Taken        : {int(minutes)} minutes and {seconds:.2f} seconds")
print("========================================================")
