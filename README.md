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
## 4. Missing Encoding
Inspect the broken image. Find the src link: 
```
http://localhost:3000/assets/public/images/uploads/😼-#zatschi-#whoneedsfourlegs-1572600969477.jpg
```
Visit this link in your browser. Then, check the Network tab. It will show that the actual visited links reads: 
```
http://localhost:3000/assets/public/images/uploads/%F0%9F%98%BC-
``` 
which is everything up until the first `#` symbol. The website can render unencoded pound signs, but your browser cannot. Encode each `#` as `%23` to get the solution URL: 
```
http://localhost:3000/assets/public/images/uploads/😼-%23zatschi-%23whoneedsfourlegs-1572600969477.jpg
```
