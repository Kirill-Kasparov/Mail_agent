# Mail_agent
Этот проект представляет собой автоматический почтовый агент , который обрабатывает входящие письма и отправляет ответы, используя модель GPT для генерации текста.

Предпосылки:
---------
После пножества проектов ботов в Телеграме, автоматизация почты давно просилась, как логичное решение.

Ключевые задачи проекта: 
- настроить устойчивую связь агента для получения и отправки сообщений.
- решить проблему кодировки, а местами даже шифрования почты.
- учесть, что не все письма нужно обрабатывать.
- дать возможность пользователю управлять настройками агента.

Внезапное решение:
---------
Обработку почты через GPT настроил исключительно для теста, чтобы имитировать отправку ответов различного формата. 

Десятки отправленных сообщений для отладки кода открыли глаза - это готовое решение для личного ассистента, которое не блокируется корпоративными настройками безопасности! Главное, отправлять обезличенные запросы.

Задачи проетка я конечно решил, но и получил удобного помощника :)



Основные функции
---------
Загрузка настроек из файла

Файл settings.txt используется для хранения конфиденциальных данных (серверы IMAP/SMTP, логин, пароль, API-ключ для GPT).
Если файл отсутствует, программа создает его шаблоном настроек.

Декодирование заголовков и текста писем

decode_mime_words(s): Расшифровывает кодированные MIME-заголовки писем.
safe_decode(payload): Пытается декодировать традиционные письма разными методами, включая UTF-8, KOI8-R, Latin-1 и другие.
Очистка текста от лишних символов

get_gpt_response(prompt): Отправляет запрос в API GPT с заданной подсказкой и возвращает текстовый ответ.
Поддерживается кастомизация модели, температуры и URL API.
Преобразование текста Markdown в HTML

markdown_to_html(md_text): Преобразует ответ из GPT из Markdown в формате HTML для отправки по электронной почте.
Обработка почты и отправка ответов

Подключение к IMAP-серверу для поиска новых (непрочитанных) писем.
Проверка отправителя на соответствие списку ALLOWED_SENDERS.
Извлечение текста из писем и его очистка.
Генерация ответа с помощью GPT, добавление исходного текста в тело ответа.
Отправка ответа с сохранением темы письма.


```python
import imaplib    # by Kirill Kasparov, 2025
import smtplib
from email import message_from_bytes
from email.header import decode_header
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.utils import parseaddr
import os
import time
import requests
import json
import markdown
import re

# Загрузка настроек из файла
SETTINGS_FILE = 'settings.txt'

def load_settings(filename):
    settings = {}
    if not os.path.exists(filename):
        with open(filename, 'w', encoding='utf-8') as f:
            f.write("IMAP_SERVER=imap.mail.yahoo.com\n")
            f.write("SMTP_SERVER=smtp.mail.yahoo.com\n")
            f.write("IMAP_PORT=993\n")
            f.write("SMTP_PORT=465\n")
            f.write("EMAIL_ADDRESS=your_email@yahoo.com\n")
            f.write("APP_PASSWORD=your_generated_app_password\n")
            f.write("ALLOWED_SENDERS=client@yandex.ru\n")
            f.write("API_GPT=sk-1234567890\n")
            f.write("API_URL=https://api.proxyapi.ru/openai/v1/chat/completions\n")
            f.write("MODEL=gpt-4o\n")
            f.write("TEMPERATURE=0.7\n")
        print(f"Файл настроек '{filename}' создан. Заполните его перед запуском.")
        exit()
    with open(filename, 'r', encoding='utf-8') as f:
        for line in f:
            key, value = line.strip().split('=', 1)
            if key == 'ALLOWED_SENDERS':
                settings[key] = set(value.split(','))
            else:
                settings[key] = value
    return settings

settings = load_settings(SETTINGS_FILE)

IMAP_SERVER = settings['IMAP_SERVER']
SMTP_SERVER = settings['SMTP_SERVER']
IMAP_PORT = int(settings['IMAP_PORT'])
SMTP_PORT = int(settings['SMTP_PORT'])
EMAIL_ADDRESS = settings['EMAIL_ADDRESS']
APP_PASSWORD = settings['APP_PASSWORD']
ALLOWED_SENDERS = settings['ALLOWED_SENDERS']
API_GPT = settings['API_GPT']
API_URL = settings['API_URL']
MODEL = settings['MODEL']
TEMPERATURE = float(settings['TEMPERATURE'])

# Инструкции из ПРОМПТ
file_path = 'gpt_prompt.txt'
if not os.path.exists(file_path):
    with open(file_path, 'w', encoding='utf-8') as file:
        pass
with open(file_path, 'r', encoding='utf-8') as file:
    gpt_prompt = file.read().strip()

# Проверка кодировки почты
def decode_mime_words(s):
    decoded_words = decode_header(s)
    decoded_string = ''
    for word, encoding in decoded_words:
        if isinstance(word, bytes):
            decoded_string += word.decode(encoding or 'utf-8')
        else:
            decoded_string += word
    return decoded_string

# Перебираем все виды кодировок
def safe_decode(payload, default_charset='utf-8'):
    try:
        # Пытаемся декодировать с помощью стандартной кодировки UTF-8
        return payload.decode(default_charset)
    except UnicodeDecodeError:
        try:
            # Пытаемся декодировать с помощью кодировки KOI8-R
            return payload.decode('koi8-r')
        except UnicodeDecodeError:
            try:
                # Пытаемся декодировать с помощью кодировки Latin-1
                return payload.decode('latin1')
            except UnicodeDecodeError:
                try:
                    # Пытаемся декодировать с помощью Quoted-Printable
                    import quopri
                    return quopri.decodestring(payload).decode(default_charset, errors='ignore')
                except UnicodeDecodeError:
                    try:
                        # Пытаемся декодировать с помощью Base64
                        import base64
                        return base64.b64decode(payload).decode(default_charset, errors='ignore')
                    except Exception:
                        # В крайнем случае используем игнорирование ошибок
                        return payload.decode('latin1', errors='ignore')

# Удаляет лишние символы, отступы и пробелы
def clean_text(text):
    # Удалить последовательные пробельные символы, заменяя их на один пробел
    cleaned_text = re.sub(r'\s+', ' ', text)  # Заменить много пробелов одним
    # Удалить непечатаемые символы, кроме букв, цифр, пробелов и знаков пунктуации
    cleaned_text = re.sub(r'[^\w\s.,;!?@#%&*()\[\]{}\'"<>\-+=:]', '', cleaned_text, flags=re.UNICODE)
    return cleaned_text.strip()

# Запрос в GPT
def get_gpt_response(prompt):
    headers = {
        'Authorization': f'Bearer {API_GPT}',
        'Content-Type': 'application/json'
    }
    data = {
        'model': MODEL,
        'messages': [{'role': 'user', 'content': gpt_prompt + prompt}],
        'temperature': TEMPERATURE
    }
    response = requests.post(API_URL, headers=headers, data=json.dumps(data))
    if response.status_code == 200:
        return response.json()['choices'][0]['message']['content']
    else:
        return 'Ошибка получения ответа от GPT.'

# Обработка запроса из формата Markdown в html для почты
def markdown_to_html(md_text):
    html_content = markdown.markdown(md_text, extensions=['extra', 'sane_lists', 'tables'])
    return html_content

# Удаляем html теги
def remove_html_tags(text):
    clean_text = re.sub(r'<[^>]*>', '', text)  # Удаляет все HTML теги
    return clean_text

print("Программа запущена...")
while True:
    # Подключение к IMAP
    with imaplib.IMAP4_SSL(IMAP_SERVER, IMAP_PORT) as mail:
        mail.login(EMAIL_ADDRESS, APP_PASSWORD)
        mail.select('inbox')

        status, messages = mail.search(None, 'UNSEEN')
        if status == 'OK':
            for msg_num in messages[0].split():
                status, data = mail.fetch(msg_num, '(RFC822)')
                if status != 'OK':
                    continue

                raw_email = data[0][1]
                email_message = message_from_bytes(raw_email)

                sender = parseaddr(email_message['From'])[1]
                subject = decode_mime_words(email_message['Subject'] or 'Без темы')
                date = email_message['Date'] or 'Неизвестная дата'
                body = ""

                ALLOWED_SENDERS = {email.strip() for email in settings['ALLOWED_SENDERS']}

                if sender in ALLOWED_SENDERS:
                    if email_message.is_multipart():
                        for part in email_message.walk():
                            content_type = part.get_content_type()
                            content_disposition = str(part.get("Content-Disposition"))
                            if content_type == "text/plain" and "attachment" not in content_disposition:
                                body = safe_decode(part.get_payload(decode=True))
                                break
                            elif content_type == "text/html" and "attachment" not in content_disposition:
                                body = safe_decode(part.get_payload(decode=True))
                    else:
                        body = safe_decode(email_message.get_payload(decode=True))

                    # Чистим входящее от мусора и html тегов
                    cleaned_body = clean_text(body)
                    body_clean = remove_html_tags(cleaned_body)
                    # Получаем ответ GPT
                    response_from_gpt = get_gpt_response(body_clean)
                    # Форматируем ответ GPT в html
                    html_response = markdown_to_html(response_from_gpt)

                    # Отправка ответа с сохранением темы и истории
                    with smtplib.SMTP_SSL(SMTP_SERVER, SMTP_PORT) as smtp:
                        smtp.login(EMAIL_ADDRESS, APP_PASSWORD)

                        msg = MIMEMultipart()
                        msg['From'] = EMAIL_ADDRESS
                        msg['To'] = sender
                        msg['Subject'] = f'Re: {subject}'
                        reply_body = f'{html_response}<br><br>--- Исходное сообщение ---<br><strong>От:</strong> {sender}<br><strong>Дата:</strong> {date}<br><strong>Тема:</strong> {subject}<br><br>{body}'
                        msg.attach(MIMEText(reply_body, 'html'))

                        smtp.sendmail(EMAIL_ADDRESS, sender, msg.as_string())
                    print(f'Ответ отправлен на {sender}')

        mail.logout()
    time.sleep(10)
```
