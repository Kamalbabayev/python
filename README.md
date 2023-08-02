import requests
import json
import pyodbc
from datetime import datetime
import tkinter as tk
import time
import keyboard
import sys
import ctypes
import socket
import win32api
import win32con

def check_internet_connection():
    try:
        # DNS sorgusu yaparak bir internet sitesine bağlanmaya çalışalım (örneğin google.com)
        socket.gethostbyname('www.google.com')
        print("İnternet bağlantısı var.")
        return True
    except socket.error:
        print("İnternet bağlantısı yok.")
        return False

# İnternet bağlantısını kontrol et
check_internet_connection()



# Check if the program is already running
mutex = ctypes.windll.kernel32.CreateMutexA(None, False, 'Mobeset')
if ctypes.windll.kernel32.GetLastError() == 183:
    sys.exit(0)


def validate_paid_by_bonus(p):
    if p == '':
        return True
    try:
        float(p)
        return True
    except ValueError:
        return False

def check_paid_by_bonus():
    paid_by_bonus = paid_by_bonus_entry.get()
    if paid_by_bonus == '':
        paid_by_bonus = 0
    else:
        paid_by_bonus = float(paid_by_bonus)
    if paid_by_bonus > active_bonus:
        paid_by_bonus_entry.configure(bg="#FFC0CB")
        send_button.configure(state='disabled')
    else:
        paid_by_bonus_entry.configure(bg="#FFFFFF")
        update_send_button_state()
    update_total()

def validate_cash_payment(p):
    if p == '':
        return True
    try:
        float(p)
        return True
    except ValueError:
        return False

def check_cash_payment():
    cash_payment = cash_payment_entry.get()
    if cash_payment == '':
        cash_payment = 0
    else:
        cash_payment = float(cash_payment)
    update_total()

def update_total():
    goods_price = float(goods_price_entry.get())
    paid_by_bonus = paid_by_bonus_entry.get()
    if paid_by_bonus == '':
        paid_by_bonus = 0
    else:
        paid_by_bonus = float(paid_by_bonus)
    cash_payment = cash_payment_entry.get()
    if cash_payment == '':
        cash_payment = 0
    else:
        cash_payment = float(cash_payment)
    total = goods_price - paid_by_bonus - cash_payment
    total = round(total, 2)
    total_entry.configure(state='normal')
    total_entry.delete(0, tk.END)
    total_entry.insert(0, total)
    total_entry.configure(state='readonly')


def validate_pin(p):
    if p == '':
        return True
    try:
        int(p)
        return True
    except ValueError:
        return False

def check_pin():
    pin = pin_entry.get()
    if pin == '':
        pin = 0
    else:
        pin = int(pin)

    # Get correct pin from database
    select_query = '''SELECT TOP 1 pin FROM Ayarlar'''
    cursor.execute(select_query)
    correct_pin = cursor.fetchone()[0]

    print(f"Girilen pin: {pin}")
    print(f"Doğru pin: {correct_pin}")

    if str(pin) != str(correct_pin):
        print("Pin yanlış")
        pin_entry.configure(bg="#FFC0CB")
        send_button.configure(state='disabled')
    else:
        print("Pin doğru")
        pin_entry.configure(bg="#FFFFFF")
        update_send_button_state()

def update_send_button_state():
    select_query = '''SELECT TOP 1 status FROM SatisIslemleri ORDER BY ID DESC'''
    cursor.execute(select_query)
    status = cursor.fetchone()[0]

    if status == 1:
        send_button.configure(state='disabled')
        notification_label.config(text="Musteri Kart melumati sefdir.", fg="green")
        customer_name_label.config(text="Müşteri adı: ")
        active_bonus_label.config(text="Aktiv bonus: ")
        passive_bonus_label.config(text="Passif bonus: ")
        # Set the goods_price_entry value to 0
        goods_price_entry.configure(state='normal')
        goods_price_entry.delete(0, tk.END)
        goods_price_entry.insert(0, 0)
        goods_price_entry.configure(state='readonly')
        # Set the total_entry value to 0
        total_entry.configure(state='normal')
        total_entry.delete(0, tk.END)
        total_entry.insert(0, 0)
        total_entry.configure(state='readonly')
    else:
        send_button.configure(state='normal')
        notification_label.config(text="")  # Bildirimi temizle



def send_data(cursor, cnxn):
    # Get paidByBonus and goodsPrice values
    paid_by_bonus = paid_by_bonus_entry.get()
    if paid_by_bonus == '':
        paid_by_bonus = 0
    else:
        paid_by_bonus = float(paid_by_bonus)
    goods_price = float(goods_price_entry.get())
    
    # Get cashPayment value
    cash_payment = cash_payment_entry.get()
    if cash_payment == '':
        cash_payment = 0
    else:
        cash_payment = float(cash_payment)
    
    # Insert paidByBonus and goodsPrice into BuySettings table
    date = datetime.now().strftime('%Y-%m-%d')
    insert_query = '''INSERT INTO BuySettings (PaidByBonus, GoodsPrice, TarihveSaat) VALUES (?, ?, ?)'''
    cursor.execute(insert_query, paid_by_bonus, goods_price, date)
    cnxn.commit()

    # ... (rest of the function)

    # POST request
    url = 'https://mobeset.az//mapi/Token'
    data = {
        "Username":"994553777766",
        "Password":"777766",
        "Source":3,
        "Version":1
    }
    response = requests.post(url, json=data)
    result = json.loads(response.text)
    token = result['token']

    # Get card number from database
    select_query = '''SELECT TOP 1 KartNumarasi FROM KartBilgileri ORDER BY ID DESC'''
    cursor.execute(select_query)
    card_number = cursor.fetchone()[0]

    # Get BonusAmount from database
    select_query = '''SELECT TOP 1 BonusAmount FROM SatisIslemleri ORDER BY ID DESC'''
    cursor.execute(select_query)
    bonus_amount = float(cursor.fetchone()[0])

    # Sell Goods request
    sell_goods_url = 'https://mobeset.az/mapi/Integration/SellGoods'
    sell_goods_data = {
        #"GoodsPrice": goods_price,
        "GoodsPrice": bonus_amount,
        "PaidByBonus": paid_by_bonus,
        "Pin": correct_pin,
        "CardNumber": card_number
    }
    #sell_goods_headers = {'Authorization': f'Bearer {token}'}
    #sell_goods_response = requests.post(sell_goods_url, json=sell_goods_data, headers=sell_goods_headers)
    #sell_goods_result = json.loads(sell_goods_response.text)
#----------------------------------------
    sell_goods_headers = {'Authorization': f'Bearer {token}'}
    sell_goods_response = requests.post(sell_goods_url, json=sell_goods_data, headers=sell_goods_headers)
    print(f"sell_goods_response.text: {sell_goods_response.text}")
    print(f"sell_goods_response.status_code: {sell_goods_response.status_code}")
    print(f"sell_goods_response.reason: {sell_goods_response.reason}")
    sell_goods_result = json.loads(sell_goods_response.text)
    print(f"Bilgi gönderildi: {sell_goods_result}")
    

    # Check if the sale was successful
    if sell_goods_result['code'] == 0:
        # Disable the send button
        send_button.configure(state='disabled')


    # Update status in SatisIslemleri table
    update_query = '''UPDATE SatisIslemleri SET status = 1 WHERE ID = (SELECT MAX(ID) FROM SatisIslemleri)'''
    cursor.execute(update_query)
    cnxn.commit()



    # Add this line to hide the form after sending data
    root.withdraw()
    





# Create the main window
root = tk.Tk()
root.title("Mobeset")
root.geometry("300x534")
root.configure(bg="#F0F0F0")
root.resizable(False, False)
root.attributes("-topmost", True)



#SQL connection
#server = 'SERVERY\MOBESET'
#database = 'Kassa1'
#username = 'sa'
#password = 'sa123321sa'

cnxn =  pyodbc.connect('Driver={SQL Server};'
                       'Server=Kamal\SQLEXPRESS;'
                       'Database=Kassa1;'
                       'UID=sa;'
                       'PWD=sa123321sa;')#pyodbc.connect('DRIVER={SQL Server};SERVER='+server+';DATABASE='+database+';UID='+username+';PWD='+ password)
#conn = pyodbc.connect('Driver={SQL Server};'
#                      'Server=SERVERY;'
#                      'Database=Kassa1;'
#                      'UID=sa;'
#                      'PWD=sa123321sa;')

cursor = cnxn.cursor()


# Get customer info from database
select_query = '''SELECT TOP 1 Username, AktivBonus, DeaktivBonus FROM KartBilgileri ORDER BY ID DESC'''
cursor.execute(select_query)
result = cursor.fetchone()
if result is not None:
    customer_name = result[0]
    active_bonus = result[1]
    passive_bonus = result[2]
else:
    # Handle the case where the query returns no results
    print("Sorgu sonucu bulunamadı")




def update_line():
    # İnternet bağlantısını kontrol et
    if check_internet_connection():
        color = "green"
    else:
        color = "red"

    # Dairenin rengini güncelle
    canvas.itemconfig(1, fill=color)

    # 1 saniye sonra tekrar çağır
    root.after(1000, update_line)

# Daireyi gösterecek Canvas widget'ını oluştur
canvas = tk.Canvas(root, width=20, height=20, bg="#F0F0F0", highlightthickness=0)
canvas.pack()

# İnternet bağlantısını kontrol et
if check_internet_connection():
    color = "green"
else:
    color = "red"

# Daireyi çiz
canvas.create_oval(2, 2, 18, 18, fill=color, outline="")

# İlk güncellemeyi yap
update_line()



# Create the customer info labels
customer_name_label = tk.Label(root, text=f"Müşteri adı: {customer_name}", font=("Arial", 12), bg="#F0F0F0")
customer_name_label.pack(pady=10)
active_bonus_label = tk.Label(root, text=f"Aktiv bonus: {active_bonus}", font=("Arial", 12), bg="#F0F0F0")
active_bonus_label.pack()
passive_bonus_label = tk.Label(root, text=f"Passif bonus: {passive_bonus}", font=("Arial", 12), bg="#F0F0F0")
passive_bonus_label.pack()

# Create the notification label
notification_label = tk.Label(root, text="", font=("Arial", 12), fg="green", bg="#F0F0F0")
notification_label.pack()

# Get goodsPrice from database
select_query = '''SELECT TOP 1 totalCem, status FROM SatisIslemleri ORDER BY ID DESC'''
cursor.execute(select_query)
result = cursor.fetchone()
goods_price = result[0]
status = result[1]

# Create the card number label and entry
card_number_label = tk.Label(root, text="Kart nömrəsi:", font=("Arial", 12), bg="#F0F0F0")
card_number_label.pack(pady=10)
card_number_entry = tk.Entry(root, font=("Arial", 12))
card_number_entry.pack()


# Create the goodsPrice label and entry
goods_price_label = tk.Label(root, text="Ödənilməli məbləği:", font=("Arial", 12), bg="#F0F0F0")
goods_price_label.pack(pady=10)
goods_price_entry = tk.Entry(root, font=("Arial", 12))
goods_price_entry.insert(0, goods_price)
goods_price_entry.configure(state='readonly')
goods_price_entry.pack()

# Create the paidByBonus label and entry
paid_by_bonus_label = tk.Label(root, text="Bonusla ödəniş:", font=("Arial", 12), bg="#F0F0F0")
paid_by_bonus_label.pack(pady=10)
vcmd1 = (root.register(validate_paid_by_bonus), '%P')
paid_by_bonus_entry = tk.Entry(root, font=("Arial", 12), validate='key', validatecommand=vcmd1)
paid_by_bonus_entry.bind('<KeyRelease>', lambda e: check_paid_by_bonus())
paid_by_bonus_entry.focus_set()
paid_by_bonus_entry.pack()

# Create the cashPayment label and entry
cash_payment_label = tk.Label(root, text="Nəğd ödəniş:", font=("Arial", 12), bg="#F0F0F0")
cash_payment_label.pack(pady=10)
vcmd3 = (root.register(validate_cash_payment), '%P')
cash_payment_entry = tk.Entry(root, font=("Arial", 12), validate='key', validatecommand=vcmd3)
cash_payment_entry.bind('<KeyRelease>', lambda e: check_cash_payment())
cash_payment_entry.pack()


# paid_by_bonus_entry giriş alanında Enter tuşuna basıldığında çağrılacak fonksiyon
def on_paid_by_bonus_entry_return(event):
    # cash_payment_entry giriş alanına odaklan
    cash_payment_entry.focus_set()

# paid_by_bonus_entry giriş alanına Return tuşu olayını bağla
paid_by_bonus_entry.bind('<Return>', on_paid_by_bonus_entry_return)

# cash_payment_entry giriş alanında Enter tuşuna basıldığında çağrılacak fonksiyon
def on_cash_payment_entry_return(event):
    # send_button düğmesine tıkla
    send_button.invoke()

# cash_payment_entry giriş alanına Return tuşu olayını bağla
cash_payment_entry.bind('<Return>', on_cash_payment_entry_return)



# Create the total label and entry
total_label = tk.Label(root, text="Total:", font=("Arial", 12), bg="#F0F0F0")
total_label.pack(pady=10)
total_entry = tk.Entry(root, font=("Arial", 12))
total_entry.configure(state='readonly')
total_entry.pack()
update_total()


# Create the pin label and entry
pin_label = tk.Label(root, text="pin:", font=("Arial", 12), bg="#F0F0F0")
pin_entry = tk.Entry(root, font=("Arial", 12))
vcmd2 = (root.register(validate_pin), '%P')
pin_entry.configure(validate='key', validatecommand=vcmd2)
pin_entry.bind('<KeyRelease>', lambda e: check_pin())
pin_label.pack_forget()
pin_entry.pack_forget()

# Get correct pin from database
select_query = '''SELECT TOP 1 pin FROM Ayarlar'''
cursor.execute(select_query)
correct_pin = cursor.fetchone()[0]

# Create the send button
send_button = tk.Button(root, text="Gönder", font=("Arial", 12), command=lambda: send_data(cursor, cnxn))
if status == 1:
    send_button.configure(state='disabled')
send_button.pack(pady=10)


# send_button düğmesine tıklandığında çağrılacak fonksiyonu değiştir
def on_send_button_click(cursor, cnxn):
    # Verileri gönder
    send_data(cursor, cnxn)
    
    # Pencereyi gizle
    root.withdraw()
    
    # HIDE tablosundaki status değerini 0 olarak güncelle
    update_query = '''UPDATE HIDE SET status = 0 WHERE ID = 1'''
    cursor.execute(update_query)
    cnxn.commit()
    # check_status fonksiyonunu tekrar çağır
    check_status()

# send_button düğmesinin command özniteliğini değiştir
send_button.configure(command=lambda: on_send_button_click(cursor, cnxn))


# Bind the Ctrl+Alt+F5 key press event
new_window = None

def check_status():
    global new_window
    
    # Eğer form görünürse, kontrol yapma
    if root.state() == 'normal':
        return
    
    # HIDE tablosundan status değerini al
    select_query = '''SELECT status FROM HIDE WHERE ID = 1'''
    cursor.execute(select_query)
    status = cursor.fetchone()[0]

    # Eğer status 0 ise pencereyi gizle
    if status == 0:
        root.withdraw()
    elif status == 1:
        # Pencereyi göster
        root.deiconify()
        # Imleci paid_by_bonus_entry giriş alanına taşı
        paid_by_bonus_entry.focus_set()
        
        # Get customer info from database
        select_query = '''SELECT TOP 1 Username, AktivBonus, DeaktivBonus FROM KartBilgileri ORDER BY ID DESC'''
        cursor.execute(select_query)
        result = cursor.fetchone()
        if result is not None:
            customer_name = result[0]
            active_bonus = result[1]
            passive_bonus = result[2]
        else:
            # Handle the case where the query returns no results
            print("Sorgu sonucu bulunamadı")

        # Update the customer info labels
        customer_name_label.config(text=f"Müşteri adı: {customer_name}")
        active_bonus_label.config(text=f"Aktiv bonus: {active_bonus}")
        passive_bonus_label.config(text=f"Passif bonus: {passive_bonus}")
        
        # Get goodsPrice from database
        select_query = '''SELECT TOP 1 totalCem, status FROM SatisIslemleri ORDER BY ID DESC'''
        cursor.execute(select_query)
        result = cursor.fetchone()
        goods_price = result[0]
        status = result[1]
        
        # Update the goodsPrice entry
        goods_price_entry.configure(state='normal')
        goods_price_entry.delete(0, tk.END)
        goods_price_entry.insert(0, goods_price)
        goods_price_entry.configure(state='readonly')
        
        # Clear the paidByBonus and cashPayment entries
        paid_by_bonus_entry.delete(0, tk.END)
        cash_payment_entry.delete(0, tk.END)
        
        # Update the total entry
        update_total()
        
        # Update the send button state
        update_send_button_state()
        
        # HIDE tablosundaki status değerini 2 olarak güncelle
        update_query = '''UPDATE HIDE SET status = 2 WHERE ID = 1'''
        cursor.execute(update_query)
        cnxn.commit()
        #crusoru paid by bonusa ekliyor
        update_send_button_state()
    elif status == 3:
        # Yeni bir pencere oluştur (eğer zaten açık değilse)
        if new_window is None or not tk.Toplevel.winfo_exists(new_window):
            new_window = tk.Toplevel(root)
            new_window.title("Uyarı")
            new_window.geometry("300x100")
            new_window.configure(bg="#F0F0F0")
            new_window.resizable(False, False)
            #crusoru paid by bonusa ekliyor
            update_send_button_state()
            
            # Uyarı mesajını göster
            warning_label = tk.Label(new_window, text="+994514266329 numarasıyla iletişime geçin. Lisans durduruldu.", font=("Arial", 12), fg="red", bg="#F0F0F0")
            warning_label.pack(pady=10)

    # Belirli bir zaman aralığında tekrar çağır (örneğin her 0.5 saniyede bir)
    root.after(500, check_status)






# İlk kontrolü yap
check_status()


# İlk kontrolü yap
check_status()

    # Get customer info from database
select_query = '''SELECT TOP 1 Username, AktivBonus, DeaktivBonus FROM KartBilgileri ORDER BY ID DESC'''
cursor.execute(select_query)
result = cursor.fetchone()

# Kontrol et: Sonuç kümesi boş olabilir mi?
if result is not None:
    customer_name = result[0]
    active_bonus = result[1]
    passive_bonus = result[2]
else:
    # Handle the case where the query returns no results
    print("Sorgu sonucu bulunamadı")

 
    # Update the customer info labels
    customer_name_label.config(text=f"Müşteri adı: {customer_name}")
    active_bonus_label.config(text=f"Aktiv bonus: {active_bonus}")
    passive_bonus_label.config(text=f"Passif bonus: {passive_bonus}")
    
    # Get goodsPrice from database
    select_query = '''SELECT TOP 1 totalCem, status FROM SatisIslemleri ORDER BY ID DESC'''
    cursor.execute(select_query)
    result = cursor.fetchone()
    goods_price = result[0]
    status = result[1]
    
    # Update the goodsPrice entry
    goods_price_entry.configure(state='normal')
    goods_price_entry.delete(0, tk.END)
    goods_price_entry.insert(0, goods_price)
    goods_price_entry.configure(state='readonly')
    
    # Clear the paidByBonus and cashPayment entries
    paid_by_bonus_entry.delete(0, tk.END)
    cash_payment_entry.delete(0, tk.END)
    
    # Update the total entry
    update_total()
    
    # Update the send button state
    update_send_button_state()

#-------------------------------------------------------------------------------------------




# Create the notification label
notification_label = tk.Label(root, text="", font=("Arial", 12), fg="green", bg="#F0F0F0")
notification_label.pack()
update_send_button_state()


# Handle the WM_DELETE_WINDOW event
def on_closing():
    #pencereyi gizle
    root.withdraw()

    # HIDE tablosundaki status değerini 0 olarak güncelle
    update_query = '''UPDATE HIDE SET status = 0 WHERE ID = 1'''
    cursor.execute(update_query)
    cnxn.commit()

    # check_status fonksiyonunu tekrar çağır
    check_status()


root.protocol("WM_DELETE_WINDOW", on_closing)

# Run the main loop
root.mainloop()

