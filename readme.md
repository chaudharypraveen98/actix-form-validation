#  Form Validation in Actix Web

Form validation is the important part of Software Development Lifecycle Process. It based on the data type define while creating a form. It basically ensures few things : -
* Prevent form abuse from malicious users.
* Reduce spams.
* Ensure form users enter only legitimate data.

## Tools Used : -
1. [Actix-Web](https://actix.rs/) : A powerful, pragmatic, and extremely fast web framework for Rust
2. [Validator](https://github.com/Keats/validator) : Macros 1.1 custom derive to simplify struct validation inspired by [marshmallow](http://marshmallow.readthedocs.io/en/latest/) and
[Django validators](https://docs.djangoproject.com/en/1.10/ref/validators/)
3. [lazy_static](https://github.com/rust-lang-nursery/lazy-static.rs) : A macro for declaring lazily evaluated statics in Rust.
4. [regex](https://github.com/rust-lang/regex) : An implementation of regular expressions for Rust.
   
### Dependencies
'./Cargo.toml'
```
[dependencies]
actix-web = "4"
validator = { version = "0.12", features = ["derive"] }
serde = { version = "1.0.104", features = ["derive"] }
lazy_static = "1.4.0"
regex = "1.5.6"
```
## Let's get Rusty

### Initializing the Rust Project
Start a new project with following `cargo new <file-name>`.

### Implementing basic actix server
Let's clone the sample code from [actix official docs](https://actix.rs/).

```
use actix_web::{get, web, App, HttpServer, Responder};

#[get("/hello/{name}")]
async fn greet(name: web::Path<String>) -> impl Responder {
    format!("Hello {name}!")
}

#[actix_web::main] // or #[tokio::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/hello", web::get().to(|| async { "Hello World!" }))
            .service(greet)
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

### Creating Struct to hold data
In this project, we are creating a student model/struct.

```
pub struct Student {
    pub name: String,
    pub email: String,
    pub age: u32,
    pub username: String,
    pub password: String,
}
```

### Implementing Serde Serialize and Deserailize traits
It will convert the json data to struct and vice versa.

```
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
pub struct Student {
    pub name: String,
    ....
}
```

You can read more [here](https://serde.rs/)

### Creating Route and Handler 
It will be a post route which accepts json form data, validate it and return response depending on whether the validation is successful or failed.

#### Route

['./src/main.rs]
```
async fn main() -> std::io::Result<()> {
    ....
    HttpServer::new(|| {
        App::new()
            .route("/", web::post().to(create_student))
    })
    ....
}
```

['./src/main.rs]
#### Handler
```
pub async fn create_student(json: web::Json<Student>) -> impl Responder {
    let is_valid = json.validate();
    match is_valid {
        Ok(_) => HttpResponse::Ok().json("success"),
        Err(err) => HttpResponse::BadRequest().json(err),
    }
}
```

Here the line `let is_valid = json.validate();` implements the validator. Now lets define the define validation rules.

### Creating validation rules

#### Lets derive the [Validator trait](https://docs.rs/validator/0.15.0/validator/#) from validator.rs.

Add the validator trait along with serde Serialize and Deserialize traits.

```
use validator::Validate;

#[derive(Serialize, Deserialize, Validate)]
pub struct Student {
    ....
}
```

#### But what are the requirements for Student Struct.

1. **Name** : It must be greater than 3 characters.
2. **Email** : It must be valid Gmail address.
3. **Age** :  It must be between 18 to 22 years.
4. **Password** : It must be 8 character with min one lower case alphabet,min one upper case alphabet, min one number and min one special character. Doesn't have white spaces.
5. **Username** : It must be alphanumeric and 6 characters long.

#### Implementing conditions
1. **Name** :
    ```
    #[validate(length(min = 3,message = "Name must be greater than 3 chars"))]
    pub name: String
    ```
    **length** : It ensures that the length of data must be greater than or less than defined length.

    **message** : It is that client side message which shows the error.
2. **Email** :
    ```
    #[validate(
        email,
        contains(pattern = "gmail", message = "Email must be valid gmail address")
    )]
    pub email: String,
    }
    ```
    **email** : It ensures that the email contains which requirement like @ symbol,domain name and extension like .com and user-id.

    **contains** and **pattern** : It ensures that the particular string must contain particular sub word(pattern) and here is gmail.

3. **Age** : 
   ```
    #[validate(range(min = 18, max = 22, message = "Age must be between 18 to 22"))]
    pub age: u32,
   ```

   **range** : It enforces that the size should fall in a particular range.

4. **Username** :
   ```
   lazy_static! {
        static ref RE_USER_NAME: Regex = Regex::new(r"^[a-zA-Z0-9]{6,}$").unwrap();
    }
    pub struct Student {
        ....
        #[validate(
            regex(
                path = "RE_USER_NAME",
                message = "Username must number and alphabets only and must be 6 characters long"
            )
        )]
        pub username: String,
        ....
    }
   ```

   **[lazy_static](https://github.com/rust-lang-nursery/lazy-static.rs)** : Lazy evaluated statics in Rust. It is executed at the runtime includes anything requiring heap allocations, like vectors or hash maps etc

   **regex** : It requires a path/pattern to match. To learn more about the regex pattern used , you can refer to this [article](https://dev.to/ziishaned/learn-regex-the-easy-way-c4g).

5. **Password** :
   ```
   lazy_static! {
        static ref RE_SPECIAL_CHAR: Regex = Regex::new("^.*?[@$!%*?&].*$").unwrap();
    }

    fn validate_password(password: &str) -> Result<(), ValidationError> {
        let mut has_whitespace = false;
        let mut has_upper = false;
        let mut has_lower = false;
        let mut has_digit = false;

        for c in password.chars() {
            has_whitespace |= c.is_whitespace();
            has_lower |= c.is_lowercase();
            has_upper |= c.is_uppercase();
            has_digit |= c.is_digit(10);
        }
        if !has_whitespace && has_upper && has_lower && has_digit && password.len() >= 8 {
            Ok(())
        } else {
            return Err(ValidationError::new("Password Validation Failed"));
        }
    }

   #[validate(
        custom(
            function = "validate_password",
            message = "Must Contain At Least One Upper Case, Lower Case and Number. Dont use spaces."
        ),
        regex(
            path = "RE_SPECIAL_CHAR",
            message = "Must Contain At Least One Special Character"
        )
    )]
    pub password: String,
   ```

   ```

   ```

   **custom** : It requires a function which implements validation and we can even add our custom message by message argument.

### Complete Code
```
extern crate lazy_static;
extern crate regex;
extern crate serde;

use actix_web::{web, App, HttpResponse, HttpServer, Responder};
use lazy_static::lazy_static;
use regex::Regex;
use serde::{Deserialize, Serialize};
use validator::{Validate, ValidationError};

lazy_static! {
    static ref RE_USER_NAME: Regex = Regex::new(r"^[a-zA-Z0-9]{6,}$").unwrap();
    static ref RE_SPECIAL_CHAR: Regex = Regex::new("^.*?[@$!%*?&].*$").unwrap();
}

#[derive(Serialize, Deserialize, Validate)]
pub struct Student {
    #[validate(length(min = 3, message = "Name must be greater than 3 chars"))]
    pub name: String,
    #[validate(
        email,
        contains(pattern = "gmail", message = "Email must be valid gmail address")
    )]
    pub email: String,
    #[validate(range(min = 18, max = 22, message = "Age must be between 18 to 22"))]
    pub age: u32,
    #[validate(
        regex(
            path = "RE_USER_NAME",
            message = "Username must number and alphabets only and must be 6 characters long"
        )
    )]
    pub username: String,
    #[validate(
        custom(
            function = "validate_password",
            message = "Must Contain At Least One Upper Case, Lower Case and Number. Dont use spaces."
        ),
        regex(
            path = "RE_SPECIAL_CHAR",
            message = "Must Contain At Least One Special Character"
        )
    )]
    pub password: String,
}
fn validate_password(password: &str) -> Result<(), ValidationError> {
    let mut has_whitespace = false;
    let mut has_upper = false;
    let mut has_lower = false;
    let mut has_digit = false;

    for c in password.chars() {
        has_whitespace |= c.is_whitespace();
        has_lower |= c.is_lowercase();
        has_upper |= c.is_uppercase();
        has_digit |= c.is_digit(10);
    }
    if !has_whitespace && has_upper && has_lower && has_digit && password.len() >= 8 {
        Ok(())
    } else {
        return Err(ValidationError::new("Password Validation Failed"));
    }
}
pub async fn create_student(json: web::Json<Student>) -> impl Responder {
    let is_valid = json.validate();
    match is_valid {
        Ok(_) => HttpResponse::Ok().json("success"),
        Err(err) => HttpResponse::BadRequest().json(err),
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    println!("Starting the server at http://127.0.0.1:8080/");
    HttpServer::new(|| App::new().route("/", web::post().to(create_student)))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}

```

### Validating
Let's spin our actix server by `cargo run`.

#### Password Validation

**Payload**

```
{
    "name":"prave",
    "email":"praveen@gmail.com",
    "age":19,
    "username":"praveee",
    "password":"123@4aA"
}
```

**Output**

<img src="images/password%20error.png" alt="password error">

#### Name Validation

**Payload**

```
{
    "name":"pr",
    "email":"praveen@gmail.com",
    "age":19,
    "username":"praveee",
    "password":"123d@4aA"
}
```

**Output**

<img src="images/name%20error.png" alt="name error">

#### Email Validation

**Payload**

```
{
    "name":"praveen",
    "email":"praveen@outlook.com",
    "age":19,
    "username":"praveee",
    "password":"123d@4aA"
}
```

**Output**

<img src="images/email%20error.png" alt="email error">

#### Age Validation

**Payload**

```
{
    "age": [
        {
            "code": "range",
            "message": "Age must be between 18 to 22",
            "params": {
                "min": 18.0,
                "value": 4,
                "max": 22.0
            }
        }
    ]
}
```

**Output**

<img src="images/age%20error.png" alt="age error">

#### Username Validation

**Payload**

```
{
    "name":"praveen",
    "email":"praveen@gmail.com",
    "age":19,
    "username":"prsdfdgfd@e",
    "password":"123d@4aA"
}
```

**Output**

<img src="images/username%20error.png" alt="username error">

Feel free to make pull request for changes and suggestion. 
