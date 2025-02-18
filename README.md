# OWASP-juice-shop-solutions
This repo holds my solutions for the OWASP Juice Shop challenges
## 1. Client-side XSS Protection
Create a new account with the malicious payload as the user's email. The application sanitizes emails in the client-side but not in the backend. Therefore, one can by-pass the client-side sanitization by submitting a POST request to http://localhost:3000/api/Users of the form:
```
{"email": "<iframe src=\"javascript:alert(`xss`)\">", "password":""}
```
Note that the inside quotes are escaped.
