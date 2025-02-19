# OWASP-juice-shop-solutions
This repo holds my solutions for the OWASP Juice Shop challenges
## 1. Confidential Document
Navigate to the "About Us" page. There is a link in the middle of the paragraph that is highlighted green. Click that link and you will be redirected to `localhost:3000/ftp/legal.md`. We want to access the content in this file's parent directory. This can be easily accomplished by changing the URL to `localhost:3000/ftp`. Open the `acquisitions.md` file to complete the challenge.
## 2. Client-side XSS Protection
Create a new account with the malicious payload as the user's email. The application sanitizes emails in the client-side but not in the backend. Therefore, one can by-pass the client-side sanitization by submitting a POST request to http://localhost:3000/api/Users of the form:
```
{"email": "<iframe src=\"javascript:alert(`xss`)\">", "password":""}
```
Note that the inside quotes are escaped.
## 3. Exposed Metrics
The linked documentation for prometheus mentions a directory called `/metrics` as the standard endpoint. Simply naviagte to `localhost:3000/metrics` to reveal metric information and complete the challenge.
