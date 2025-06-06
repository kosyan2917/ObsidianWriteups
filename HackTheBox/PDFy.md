#web #pdf #ssrf #lfi
Сайтик, который ждет от нас ссылку, в ответ генеря пдфку сайта. Использует wkhtmltopdf - база корявости. Пробуем закинуть туда локалхост, поулчаем ошибку:

```
**There was an error: Error generating PDF: Command '['wkhtmltopdf', '--margin-top', '0', '--margin-right', '0', '--margin-bottom', '0', '--margin-left', '0', 'http://localhost', 'application/static/pdfs/93adbca689a1cfb0340e674ef1e4.pdf']' returned non-zero exit status 1.**
```

Хостим свой сайтик с редиректом на file:///etc/passwd, кидаем на него ссылку, забираем флаг