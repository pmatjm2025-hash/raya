import os
import json
import gspread
from oauth2client.service_account import ServiceAccountCredentials
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from webdriver_manager.chrome import ChromeDriverManager
import time
from datetime import datetime

# Ambil link spreadsheet dari environment variable (opsional, tapi lebih aman)
LINK_SPREADSHEET = "https://docs.google.com/spreadsheets/d/1ssi1n1H2UhSGsQsBG-dV1lcaWchBSKL9FLNrkpluDOM/edit?gid=0#gid=0"
NAMA_TAB_TUNGGAL = "SCRAPING"

DAFTAR_AKUN = [
    {"email": "pmalpm_sa@pinusmerahabadi.co.id", "pass": "pmaoffice99"},
    {"email": "pmamds_sa@pinusmerahabadi.co.id", "pass": "pmaoffice99"},
    {"email": "pmamdn_sa@pinusmerahabadi.co.id", "pass": "pmaoffice99"},
    {"email": "pmamdu_sa@pinusmerahabadi.co.id", "pass": "Pmaoffice30@"}
]

def run_full_sync(akun, is_first_account):
    sekarang = datetime.now()
    waktu_log = sekarang.strftime("%Y-%m-%d %H:%M:%S")
    print(f"\n[{waktu_log}] Memulai sinkronisasi untuk akun: {akun['email']}...")

    # KONEKSI GOOGLE SHEET MENGGUNAKAN GITHUB SECRETS
    try:
        SCOPE = ["https://spreadsheets.google.com/feeds", "https://www.googleapis.com/auth/drive"]
        
        # Mengambil kredensial dari Github Secret berupa String JSON
        gcs_secret = os.environ.get("GCP_SA_KEY")
        if not gcs_secret:
            raise Exception("Environment variable 'GCP_SA_KEY' tidak ditemukan!")
            
        creds_dict = json.loads(gcs_secret)
        creds = ServiceAccountCredentials.from_json_keyfile_dict(creds_dict, SCOPE)
        client = gspread.authorize(creds)
        
        ss = client.open_by_url(LINK_SPREADSHEET)
        sheet = ss.worksheet(NAMA_TAB_TUNGGAL)
    except Exception as e:
        print(f"Gagal koneksi Sheets untuk {akun['email']}: {e}")
        return

    # SETTING CHROME UNTUK LINGKUNGAN LINUX/GITHUB ACTIONS
    options = webdriver.ChromeOptions()
    options.add_argument("--headless=new") # Menggunakan headless mode versi baru
    options.add_argument("--window-size=1920,1080")
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")
    options.add_argument("--disable-gpu")
    options.add_argument("--blink-settings=imagesEnabled=false") # Opsional: matikan gambar agar lebih cepat

    driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)
    wait = WebDriverWait(driver, 30)

    try:
        driver.get("https://eds.nabatisnack.co.id/auth/login")
        wait.until(EC.presence_of_element_located((By.NAME, "email"))).send_keys(akun['email'])
        driver.find_element(By.NAME, "password").send_keys(akun['pass'])
        driver.find_element(By.CSS_SELECTOR, "button[type='submit']").click()
        time.sleep(10)

        driver.get("https://eds.nabatisnack.co.id/delivery-order")
        
        all_collected_data = []
        halaman = 1

        while True:
            print(f"[{akun['email']}] Membaca Halaman {halaman}...")
            try:
                wait.until(EC.presence_of_element_located((By.CSS_SELECTOR, "table tbody tr td")))
                time.sleep(5) 

                rows = driver.find_elements(By.CSS_SELECTOR, "table tbody tr")
                for row in rows:
                    cols = row.find_elements(By.TAG_NAME, "td")
                    if len(cols) > 1:
                        isi_baris = [c.text.strip() for c in cols]
                        all_collected_data.append([waktu_log, akun['email']] + isi_baris)
                
                print(f"-> Berhasil mengambil {len(rows)} baris.")

                next_btn_xpath = "//li[contains(@class,'next') and not(contains(@class,'disabled'))]/a | //button[contains(.,'Next') and not(@disabled)]"
                next_btn = driver.find_elements(By.XPATH, next_btn_xpath)
                
                if next_btn and next_btn[0].is_displayed():
                    driver.execute_script("arguments[0].click();", next_btn[0])
                    halaman += 1
                    time.sleep(6)
                else:
                    break
            except Exception as e:
                print(f"Berhenti di halaman {halaman} karena: {e}")
                break

        if all_collected_data:
            if is_first_account:
                sheet.clear()
                header = ["Waktu Scraping", "Akun Pengirim", "No DO", "Tgl DO", "Customer", "Alamat", "Status", "Total"]
                final_payload = [header] + all_collected_data
                sheet.update('A1', final_payload, value_input_option='USER_ENTERED')
                print(f"Berhasil mereset sheet dan memasukkan data akun pertama ({akun['email']}).")
            else:
                sheet.append_rows(all_collected_data, value_input_option='USER_ENTERED')
                print(f"Berhasil menambahkan (append) data akun berikutnya ({akun['email']}).")
        else:
            print(f"PERINGATAN: Tidak ada data ditemukan untuk {akun['email']}.")

    except Exception as e:
        print(f"CRITICAL ERROR ({akun['email']}): {e}")
    finally:
        driver.quit()
        print(f"[{datetime.now().strftime('%H:%M:%S')}] Robot Selesai untuk {akun['email']}.")

if __name__ == "__main__":
    print("Sistem monitoring gabungan aktif di GitHub Actions.")
    hari_ini = datetime.now().weekday()  
    
    if hari_ini != 6:  # Hari Minggu libur (6 = Minggu)
        for index, akun in enumerate(DAFTAR_AKUN):
            is_first = (index == 0)
            try:
                run_full_sync(akun, is_first_account=is_first)
                time.sleep(10) 
            except Exception as e:
                print(f"Error pada siklus akun {akun['email']}: {e}")
        print("\n[INFO] Semua data akun selesai diproses pada siklus ini.")
    else:
        print(f"[{datetime.now().strftime('%H:%M:%S')}] Hari Minggu. Sistem libur otomatis.")
