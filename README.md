# E-commerce Backend

You will build the API back-end for your E-commerce store using Flask. The store front-end will be built on a separate day that utilizes the API. The store will have the following functionality:

* Listing and displaying the products
* User sign up
* User login
* Adding products to a shopping cart
* Checking out the products on the shopping cart

The back-end will support this functionality by providing a set of APIs that can be used by the front-end via AJAX requests. Your back-end will use a PostgreSQL database. A reference database schema has been provided to you, but you can change it to your liking. You will test out your APIs as you develop them using Postman.

Throughout this document, I will define APIs in the format:

```
GET /api/some_path/{some_parameter}
```

Where the first word is the request method and will either be ```GET``` or ```POST```. Then the URL path will always start with the prefix ```/api/``` to differentiate it with where the front-end is served. Then, the rest of the path is listed, and may contain URL parameters. Parameters are denoted as ```{parameter_name}```. (But of course, the syntax for Flask is different ;)


## Things to know

* ```jsonify``` - a function provided by the ```flash``` module that turns data into JSON format to be returned as the response
* ```request.get_json()``` - when a POST request is made by the client and the client is sending JSON-formatted data in the request body (as AngularJS does by default), ```request.get_json()``` is what you will need to use to get that data in dictionary form instead of ```request.form```
* To send data in JSON format request body within Postman, you will select the "raw" option within the "Body" tab of the request. Then, you will want to change "Text" on the right-most dropdown to "JSON (application/json)", and then write the JSON-formatted data in the textarea underneath
* ```uuid``` - this is a standard module that will generate long random strings for use as authentication tokens, more on this later
* returning status codes - you should return specific status codes when things go wrong to represent specific error conditions. To do this, add an addition status code number to the end of your return statement: Ex: ```return jsonify(results), 401```. Here are some status codes we will use
  * 401 - Unauthorized

```
## GET /api/products
```

This API returns information for all products in the database as an array of objects.

```
## GET /api/product/{id}
```

This API returns the information for a single product by the product ID.

```
## POST /api/user/signup
```

This API allows a new user to sign up. The request body will be in JSON format, not form-data format. This means that in Postman, in the Body tab of the request, select "raw", and "JSON (application/json)" in the dropdown to the right. You will write the body in JSON format in the text area below.

To implement a user sign up, you will add an entry to the ```customer``` table (we are using the name "customer" because "user" is a reserved word in PostgreSQL). The following fields are straightforward:
* username
* email
* first_name
* last_name
Storing passwords needs more work, because we will store passwords securely using the bcrypt method. You will first install the ```bcrypt``` module using ```pip```:

```
pip install bcrypt
```

Then, you will use it to encrypt a password like so:

```python
import bcrypt

password = 'opensesame' # the entered password
salt = bcrypt.gensalt() # generate a salt
# now generate the encrypted password
encrypted_password = bcrypt.hashpw(password.encode('utf-8'), salt)
```

Once the encrypted password is generated, that's what you want to store into the database in the customer record, not the plain text password.

```
## POST /api/user/login
```

This API handles user logins. It will verify the user's password matches (more on this in the next subsection), and if it matches, it will generate an authentication token to return to the requester, which the requester will use for up to 30 days to proof their identity. The authentication token will be stored in the ```auth_token``` table and will be associated with the logged in customer.

### Verifying the password

To verify the password, again you'll need to use the ```bcrypt``` module. Here is how you do it:

```python
password = u'opensesame' # password entered by user for login
encrypted_password = user.password  # encrypted password retrieved
                                    # from database record
# the following line will take the original salt that was used
# in the generation of the encrypted password, which is stored as
# part of the encrypted_password, and hash it with the entered password
rehash = bcrypt.hashpw(password.encode('utf-8'), encrypted_password)
# if we get the same result, that means the password was correct
if rehash == encrypted_password:
    print 'Login success!'
else:
    print 'Login failed!'
```

If the login failed, you should return an error with the status of 401 which means Unauthorized. If the login succeeds, you should return the user information along with the authentication token to the response body. Example response body:

```
{
  "user": {
    // all the user fields here
  },
  "auth_token": "f3a001f3-a501-42dd-bf37-df20162cfddc"
}
```

As to *how* to generate the authentication token, see the next section.

### Generating the authentication token

You will use the ```uuid``` module to generate the authentication token. To do this:

```
import uuid

token = uuid.uuid4()
```

That's it!


## Authenticating the user for API calls

For the rest of the APIs you are going to implement, you will need to authenticate the user before allowing them access to the resource. If the user has not proved that they have logged in with a valid authentication token, you will return a 403 status code for forbidden.

In order to validate that the user has been logged in, they have to send the authentication token into the request for each of the authenticated APIs. They will send it in either via a query parameter in the case of GET requests, i.e. ```?auth_token=f3a001f3-a501-42dd-bf37-df20162cfddc```, or within the request body as a property of the posted JSON object:
```
{
  "auth_token": "f3a001f3-a501-42dd-bf37-df20162cfddc"
}
```

You API will have to pick out this token:

* in the case of GET requests from ```request.args.get('auth_token')```
* and in the case of POST requests from ```request.get_json().get('auth_token')```
and use it to look up the database to find the user that's been logged in via this token.

```
## POST /api/shopping_cart
```

*This API requires authentication in order to access*.

Sending a POST request to the shopping cart API will add a new product to the cart. The request body will contain a JSON object containing the ID of the product to add, something like:
```
{
  "auth_token": "f3a001f3-a501-42dd-bf37-df20162cfddc",
  "product_id": 347
}
```
In response, the back-end should create a new entry in the ```product_in_shopping_cart``` table, linking the product with the customer.

GET /api/shopping_cart

*This API requires authentication in order to access*.

Sending a GET request to the shopping cart API will return the items that are currently in the shopping cart for that user as an array of objects representing the products.

POST /api/shopping_cart/checkout

*This API requires authentication in order to access*.

Sending a POST request to the shopping cart checkout API will create a purchase record in the ```purchase``` table. This will eventually support charging credit cards with the Stripe API, but we will defer that to another day.

In addition to inserting to the ```purchase``` table, we also want to record which products were purchased by linking the new purchase with each product that was bought with that purchase by adding records to the ```product_in_database``` table.

Finally, as the last step in the checkout, clear out the shopping cart for this customer by deleting each record in ```product_in_shopping_cart``` associated with this user.

```
"auth_token": "f3a001f3-a501-42dd-bf37-df20162cfddc"
```
