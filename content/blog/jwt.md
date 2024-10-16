---
title: "Building a Golang backend system with JWT authentication"
description: "When creating a website's backend, one very important term we get to hear is JWT authentication. JWT authentication is one of the most popular ways of securing APIs. JWT stands for JSON Web Token and it is an open standard that defines a way for transmitting information between parties as a JSON object and that too securely..."
dateString: October 2024
draft: false
tags: ["Golang", "Backend", "Gin", "JWT authentication"]
weight: 99
cover:
    image: "blog/jwt-logo.png"
---

When creating a website's backend, one very important term we get to hear is JWT authentication. JWT authentication is one of the most popular ways of securing APIs. JWT stands for JSON Web Token and it is an open standard that defines a way for transmitting information between parties as a JSON object and that too securely. In this article, we will discuss JWT authentication and most importantly we will create an entire project for Fast and Efficient JWT authentication using Gin-Gonic, this will be a step-by-step project tutorial that will help you create a very fast and industry-level authentication API for your website or application's backend. 

**Table of Content:**

- What is JWT authentication?
- Project Structure
- Prerequisites
- Step by Step Tutorial
- Final Results
- Output
- Conclusion

## What is JWT authentication?

JWT stands for JSON Web Token and it is an open standard that defines a way for transmitting information between parties as a JSON object and that too securely.

Let's try to understand that by the help of an example. Consider a situation when we show up at a hotel and we walk up to the front desk and the receptionist says "hey, what can I do for you?". I would say "Hi, my name is Shubham, I'm here for a conference, the sponsors are paying for my hotel". The receptionist says "okay great! well I'm going to need to see a couple things". Usually they'll need to see my identification so to prove who I am and once that they have verified that I am the right person, they will issue me a key. And authentication works in a very similar way to this example.

With JWT authentication, we are making a request out to a server saying "Hey! here's my username and password or sign-in token is this" and the website says "okay, let me check." If my username and password is correct then that would give me a token. Now on subsequent requests of the server I don't have to any longer include my username and password. I would just carry my token and check into the hotel (website), access to the Gym (data), I would have access to the pool(information) and only to my hotel room (account), I don't have access to any other hotel rooms (other user's account). This token is only authorized during the duration of my state so from check-in time to the checkout time. After that it is of no use. Now the hotel will also allow people without any verification to at least see the hotel, to roam in the public area around the hotel until you are coming inside the hotel, Similarly being an anonomous user you can interact with the website's home page, landing page, etc.

**Here's an example of a JWT :**

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ 
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

## Project Structure

Here is the structure of the project. Make sure to create a similar folder structure in your workspace as well. In this structure we have 6 folders:

1. controllers
2. database
3. helpers
4. middleware
5. models
6. routes

and create the respective files inside these folders.

![Project Structure](/blog/jwt/one.webp)

### Prerequisites

1. Go - Version 1.18+
2. Mongo DB
3. Step by Step Tutorial

**Step 1.** Let's start the project by creating a module, the name of my module is "jwt" and my username is "1shubham7", therefore I will initial my module by entering:

```sh
go mod init github.com/1shubham7/jwt
```

This will create a go.mod file.

**Step 2.** Create a main.go file and let's create a web server in main.go. For that, add the following code in the file:

```go
package main
import(
	"github.com/gin-gonic/gin"
	"os"
	routes "github.com/1shubham7/jwt/routes"
	"github.com/joho/godotenv"
	"log"
)
func main() {
	err := godotenv.Load(".env")
	if err != nil {
		log.Fatal("Error locading the .env file")
	}
	port := os.Getenv("PORT")
	if port == "" {
		port = "1111"
	}
	router := gin.New()
	router.Use(gin.Logger())
	routes.AuthRoutes(router)
	routes.UserRoutes(router)
	// creating two APIs
	router.GET("/api-1", func(c *gin.Context){
		c.JSON(200, gin.H{"success":"Access granted for api-1"})
	})
	router.GET("api-2", func(c *gin.Context){
		c.JSON(200, gin.H{"success":"Access granted for api-2"})
	})
	router.Run(":" + port)
}
```

'os' package to retrieve environment variables.
we are using gin-gonic package to create a web server.
Later we will create a routes package.
AuthRoutes(), UserRoutes() are functions inside a file from routes package, we will create it later on.

**Step 3.** Download the gin-gonic package:

```sh
go get github.com/gin-gonic/gin
```

***Step 4.*** Create a models folder and inside it create a userModel.go file. Enter the following code inside the userModel.go:

```go
package models
import (
	"go.mongodb.org/mongo-driver/bson/primitive"
	"time"
)
type User struct {
	ID            primitive.ObjectID `bson:"id"`
	First_name    *string            `json:"first_name" validate:"required, min=2, max=100"`
	Last_name     *string            `json:"last_name" validate:"required, min=2, max=100"`
	Password      *string            `json:"password" validate:"required, min=6"`
	Email         *string            `json:"email" validate:"email, required"` //validate email means it should have an @
	Phone         *string            `json:"phone" validate:"required"`
	Token         *string            `json:"token"`
	User_type     *string            `json:"user_type" validate:"required, eq=ADMIN|eq=USER"`
	Refresh_token *string            `json:"refresh_token"`
	Created_at    time.Time          `json:"created_at"`
	Updated_at    time.Time          `json:"updated_at"`
	User_id       string             `json:"user_id"`
}
```

We have created a struct called User and have added necessary fields to the struct.

`json:"first_name" validate:"required, min=2, max=100"` this are called field tags, these are used during decoding and encoding of go code to JSON and JSON to go.
Here validate:"required, min=2, max=100" this is used for validate that the particular field must have minimum 2 characters and maximum 100 characters.

**Step 5.** Create a database folder and inside it create a databaseConnection.go file, enter the following code inside it:

```go
package database
import (
	"fmt"
	"log"
	"os"
	"time"
	"context"
	"github.com/joho/godotenv"
	"go.mongodb.org/mongo-driver/mongo"
	"go/mongodb.org/mongo-driver/mongo/options"
)
func DBinstance() *mongo.Client{
	err := godotenv.Load(".env")
	if err != nil {
		log.Fatal("Error locading the .env file")
	}
	MongoDb := os.Getenv("THE_MONGODB_URL")
	client, err := mongo.NewClient(options.Client().ApplyURI(MongoDb))
	if err != nil {
		log.Fatal(err)
	}
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	err = client.Connect(ctx)
	if err!=nil{
		log.Fatal(err)
	}
	fmt.Println("Connected to MongoDB!!")
	return client
}
var Client *mongo.Client = DBinstance()
func OpenCollection(client *mongo.Client, collectionName string) *mongo.Collection {
	var collection *mongo.Collection = client.Database("cluster0").Collection(collectionName)
	return collection
}
```

also make sure to download the 'mongo' package:

```sh
go get go.mongodb.org/mongo-driver/mongo
```

Here we are connecting your mongo database with the application.

we are using 'godotenv' for loading environment variables that we will set in the .env file in main directory.
The DBinstance Function, we take the value of "THE_MONGODB_URL" from the .env file (we will create that in the coming steps) and create a new mongoDB client using the value.
'context' is used to have a timeout of 10 seconds.
OpenCollection Function() takes client and collectionName as input and creates a collection for it.

**Step 6.** For the routes, we will create two different files, authRouter and userRouter. authRouter includes '/signup' and '/login' . These will public to everyone so that they can authorize themselves. userRouter will not be public to everyone. It will include '/users' and '/users/:user_id'.

Create a folder called routes and add two files into it:

- userRouter.go
- authRouter.go

enter the following code into userRouter.go:

```go
package routes
import (
	"github.com/gin-gonic/gin"
	controllers "github.com/1shubham7/jwt/controllers"
	middleware "github.com/1shubham7/jwt/middleware"
)
// user should not be able to use userRoute without the token
func UserRoutes (incomingRoutes *gin.Engine) {
	incomingRoutes.Use(middleware.Authenticate())
	// user routes are public routes but these must be authenticated, that
	// is why we have Authenticate() before these
	incomingRoutes.GET("/users", controllers.GetUsers())
	incomingRoutes.GET("users/:user_id", controllers.GetUserById())
}
```

**Step 7.** enter the following code into the authRouter.go:

```go
package routes
import (
	"github.com/gin-gonic/gin"
	controllers "github.com/1shubham7/jwt/controllers"
)
// this is when the user has not signed up. userRouter is when the user has logged in
// and has the token.
func AuthRoutes(incomingRoutes *gin.Engine) {
	incomingRoutes.POST("users/signup", controllers.SignUp())
	incomingRoutes.POST("user/login", controllers.Login())
}
```

**Step 8.** create a folder called controllers and add a file called 'userController.go' to it. enter the following code inside it.

```go
package controllers
import (
	"context"
	"fmt"
	"log"
	"net/http"
	"strconv"
	"time"
	database "github.com/1shubham7/jwt/database"
	helper "github.com/1shubham7/jwt/helpers"
	models "github.com/1shubham7/jwt/models"
	"github.com/gin-gonic/gin"
	"github.com/go-playground/validator/v10"
	"golang.org/x/crypto/bcrypt"
)
var userCollection *mongo.Collection = database.OpenCollection(database.Client, "user")
func Hashpassword()
func VerifyPassword()
func SignUp()
func Login()
func GetUsers()
func GetUserById()

The packages used are straight forward. database, helper and models are our own packages.
'fmt' package is used for string formatting.
'time' package is used for time representation
'net/http' package is used for dealing with requests
'strconv' package is used to convert string to int and vice versa.
Then we have just created the outline function, we will complete those functions in the coming steps.
Step 9. To keep environment variables in a separate file, we will create a .env file inside the main folder (root folder) and add the following code into it:

PORT=1111
THE_MONGODB_URL=mongodb://localhost:27017/go-auth
```

These two variables have already been used in our previous code.

**Step 10.** Let's create the GetUserById() function first. Enter the following code in GetUserById() function:

```go
func GetUserById() gin.HandlerFunc{
	return func(c *gin.Context){
		userId := c.Param("user_id") // we are taking the user_id given by the user in json
		// with the help of gin.context we can access the json data send by postman or curl or user
		
		if err := helper.MatchUserTypeToUserId(c, userId); err != nil{
			//checking if the user in admin or not.
			// we will create that func in helper package.
				c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
				return 
			}
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		var user models.User
		
		err := userCollection.FindOne(ctx, bson.M{"user_id": userId}).Decode(&user)
		defer cancel()
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		// if everything goes ok, pass the data of the user (UserModel.go)
		c.JSON(http.StatusOK, user)
	}
}
```

**Step 11.** Let's create a folder called helpers and add a file called authHelper.go into it. Enter the following code into the authHelper.go :

```go
package helpers
import (
	"errors"
	"github.com/gin-gonic/gin"
)
func CheckUserType(c *gin.Context, userOrAdmin string) (err error) {
	userType := c.GetString("user_type")
	err = nil
	if userType != userOrAdmin {
		err = errors.New("Not authorized to access the resource")
		return err
	}
	return err
}
func MatchUserTypeToUserId(c *gin.Context, userId string) (err error) {
	//  This is the match user function
	userType := c.GetString("user_type")
	uid := c.GetString("uid")
	err = nil
	// this means that user is USER not ADMIN and uid is not of the user. Because user can only access his id,
	// admin can access anyone's id
	if userId == "USER" && uid != userId {
		err = errors.New("You are not authorized to access this user")
	}
	err = CheckUserType(c, userType)
	return err
}
```

The MatchUserTypeToUserId() function just matches if the user is a Admin or just a user.
We are using CheckUserType() function inside MatchUserTypeToUserId(), this is just checking if everything is fine (if the user_type we get from user is same as the userType variable.

**Step 12.** We can now work on the SignUp() function of userController.go:

```go
func SignUp()gin.HandlerFunc{
	return func(c *gin.Context){
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		var user models.User
		if err := c.BindJSON(&user); err!=nil{
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		validationErr := validate.Struct(user)
		// this is used to validate, but what? see the User struct, and see those validate struct fields
		if validationErr != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": validationErr.Error()})
			return
		}
		// we are using count to help us validate. if you find documents with the user email already
		// then count would be more than 0, and we can then handle that err
		count, err := userCollection.CountDocuments(ctx, bson.M{"email":user.Email})
		defer cancel()
		if err != nil {
			log.Panic(err)
			c.JSON(http.StatusInternalServerError, gin.H{"error":"error occured while checking for the email"})
		}
		password := HashPassword(*user.Password)
		user.Password = &password
		count, err = userCollection.CountDocuments(ctx, bson.M{"phone":user.Phone})
		defer cancel()
		if err != nil {
			log.Panic(err)
			c.JSON(http.StatusInternalServerError, gin.H{"error":"error occured while checking for the phone number"})
		}
		if count > 0 {
			c.JSON(http.StatusInternalServerError, gin.H{"error":"this email or phone number already exists"})
		}
		// by "c.BindJSON(&user)" user already have the information from the website user
		user.Created_at, _  = time.Parse(time.RFC3339, time.Now().Format(time.RFC3339))
		user.Updated_at, _  = time.Parse(time.RFC3339, time.Now().Format(time.RFC3339))
		user.ID = primitive.NewObjectID()
		user.User_id = user.ID.Hex()
		token, refreshToken, _ := helper.GenerateAllTokens(*user.Email, *user.First_name, *user.Last_name, *user.User_type, *&user.User_id)
		
		// giving value that we generated to user
		user.Token = &token
		user.Refresh_token = &refreshToken
		// now let's insert it to the database
		resultInsertionNumber, insertErr :=  userCollection.InsertOne(ctx, user)
		if insertErr != nil {
			msg := fmt.Sprintf("User item was not created")
			c.JSON(http.StatusInternalServerError, gin.H{"error": msg})
			return
		}
		defer cancel()
		c.JSON(http.StatusOK, resultInsertionNumber)
	}
}
```

- We created a variable user of type User.

- validationErr is used to validate the struct tags, we already discussed this.

- We are also using count variable to validate. As in if we find documents with the user email already then the count would be more than 0, and we can then handle that error (err)

- Then we are using 'time.Parse(time.RFC3339, time.Now().Format(time.RFC3339))' to set the time for Created_at, Updated_at struct fields. When the user tries to sign up, the function SignUp() will run and that particular time will be stored in  Created_at, Updated_at struct fields.

- Then we are creating tokens using GenerateAllTokens() function that we will create in tokenHelper.go file in the same package in the next step.

- We also have a HashPassword() function that is doing nothing but hashing the user.password and replacing the user.password with the hashed password. We will also create that thing later.

- And then we are just inserting the data and the tokens etc. to the userCollection

- If everything goes right, we will give StatusOK back.

**Step 13.** create a file in helpers folder called 'tokenHelper.go' and enter the following code inside it.

```go
package helpers
import (
	"context"
	"fmt"
	"log"
	"os"
	"time"
	"github.com/1shubham7/jwt/database"
	jwt "github.com/dgrijalva/jwt-go" // golang driver for jwt
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)
type SignedDetails struct {
	Email string
	First_name string
	Last_name string
	Uid string
	User_type string
	jwt.StandardClaims
}
var userCollection *mongo.Collection = database.OpenCollection(database.Client, "user")
// btw we sould have our secret key in .env for production 
var SECRET_KEY string = os.Getenv("SECRET_KEY")
func GenerateAllTokens(email string, firstName string, lastName string, userType string, uid string) (signedToken string, signedRefreshToken string, err error){
	claims := &SignedDetails{
		Email : email,
		First_name: firstName,
		Last_name: lastName,
		Uid : uid,
		User_type: userType,
		StandardClaims: jwt.StandardClaims{
			// setting the expiry time
			ExpiresAt: time.Now().Local().Add(time.Hour *time.Duration(120)).Unix(),
		},
	}
		// refreshClaims is used to get a new token if th eprevious once is expired.
	
		refreshClaims := &SignedDetails{
			StandardClaims: jwt.StandardClaims{
				ExpiresAt: time.Now().Local().Add(time.Hour *time.Duration(172)).Unix(),
		},
	}
	token ,err := jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString([]byte(SECRET_KEY))
	refreshToken, err := jwt.NewWithClaims(jwt.SigningMethodHS256, refreshClaims).SignedString([]byte(SECRET_KEY))
	if err!= nil{
		log.Panic(err)
		return
	}
	return token, refreshToken, err
}
```

Also make sure to download the github.com/dgrijalva/jwt-go package:

```sh
go get github.com/dgrijalva/jwt-go
```

- Here we are using github.com/dgrijalva/jwt-go for generating tokens.

- We are creating a struct called SignedDetails with field names required for generating tokens.

- we are using NewWithClaims to generate new tokens and are giving the value to the claims and refreshClaims structs. claims has token for the first time users and refreshClaims has it when the user has to refresh the token. that is, they previously had a token which is now expired.

- time.Now().Local().Add(time.Hour *time.Duration(120)).Unix() is being used for setting the expiry of the token.

- we are then simply returning three things - token, refreshToken and err which we are using in the SignUp() function.

**Step 14.** In the same file as SignUp() function, let's create the HashPassword() function we talked about in step 9.

```go
func HashPassword(password string) string {
	hashed, err:=bcrypt.GenerateFromPassword([]byte(password), 14)
	if err!=nil{
		log.Panic(err)
	}
	return string(hashed)
}
```

- We are simply using the GenerateFromPassword() method from bcrypt package which is used to generate hashed passwords from the actual password.

- This is important because we would not want and hacker to hack into our systems and steal all the passwords, also for the users privacy.

- []byte (array of bytes) is simply string.

**Step 15.** Let's create the Login() function in 'userController.go' and in later steps we can create the functions that Login() uses:

```go
func Login() gin.HandlerFunc{
	return func(c *gin.Context) {
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		var user models.User
		var foundUser models.User
		// giving the user data to user variable
		if err := c.BindJSON(&user); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		// finding the user through email and if found, storing it in foundUser variable
		err := userCollection.FindOne(ctx, bson.M{"email": user.Email}).Decode(&foundUser)
		defer cancel()
		if err!=nil{
			c.JSON(http.StatusInternalServerError, gin.H{"error": "email or password is incorrect"})
			return 
		}
		// we need pointer to acess the origina user and foundUser,
		// if we only pass user and foundUser, it will create a new instance of user and foundUser
		isPasswordValid, msg := VerifyPassword(*user.Password, *foundUser.Password)
		defer cancel()
		if isPasswordValid != true{
			c.JSON(http.StatusInternalServerError, gin.H{"error": msg})
			return
		}
		if foundUser.Email == nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": "user not found"})
		}
		token, refreshToken, _ := helper.GenerateAllTokens(*foundUser.Email, *foundUser.First_name, *foundUser.Last_name, *foundUser.User_type, foundUser.User_id)
		helper.UpdateAllTokens(token, refreshToken, foundUser.User_id)
		err = userCollection.FindOne(ctx, bson.M{"user_id":foundUser.User_id}).Decode(&foundUser)
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusOK, foundUser)
	}
}
```

- We are creating two variables of type User, these are user and Founduser. and giving the data from request to user.

- By the help of 'userCollection.FindOne(ctx, bson.M{"email": user.Email}).Decode(&foundUser)' we are finding the user through his/her email and if found, storing it in foundUser variable.

- Then we are using VerifyPassword() function to verify the password and store it, remember that we are taking pointers as parameters in VerifyPassword(), if not, it would create a new instance of those in parameters rather than actually changing them.

- We will create VerifyPassword() in the next step.

- Then we are simply using GenerateAllTokens() and UpdateAllTokens() to generate and update the token and refreshToken (which are basically tokens).

- And every step, we are all handling the errors.

**Step 16.** Let's create the VerifyPassword() function in the same file as Login() function:

```go
func VerifyPassword(userPassword, providedPassword string) (bool, string) {
	
	err := bcrypt.CompareHashAndPassword([]byte(providedPassword), []byte(userPassword))
	check := true
	msg := ""
	if err != nil {
		check = false
		msg = fmt.Sprintf("email or password is incorrect.")
	}
	return check, msg
}
```

- We are simply using the CompareHashAndPassword() method from bcrypt package which is used to compare hashed passwords. and returning a boolean value depening on the results.

- []byte (array of bytes) is simply string, but []byte helps in comparing.

**Step 17.** Let's create the UpdateAllTokens() function in the 'tokenHelper.go' file:

```go
func UpdateAllTokens(signedToken string, signedRefreshToken string, userId string){
	var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
	var updateObj primitive.D
	updateObj = append(updateObj, bson.E{"token", signedToken})
	updateObj = append(updateObj, bson.E{"refresh_token", signedRefreshToken})
	Updated_at, _ := time.Parse(time.RFC3339, time.Now().Format(time.RFC3339))
	updateObj = append(updateObj, bson.E{"updated_at", Updated_at})
	upsert := true
	filter := bson.M{"user_id":userId}
	opt := options.UpdateOptions{
		Upsert: &upsert,
	}
	_, err := userCollection.UpdateOne(
		ctx,
		filter,
		bson.D{
			{"$set", updateObj},
		},
		&opt,
	)
	defer cancel()
	if err!=nil{
		log.Panic(err)
		return
	}
	return
}
```

- We are creating a variable called updateObj which is of type primitive.D. primitive.D type in MongoDB's Go driver is a representation of a BSON document.

- Append() is appending a key-value pair to updateObj each and every time its running.

- Then 'time.Parse(time.RFC3339, time.Now().Format(time.RFC3339))' is used to update the current time (time when the updation happens and the function runs) to Updated_at.

- Rest of the code block is performing update using the UpdateOne method of mongoDB collection.

- At the last step we are also handling the error in case any error occurs.

**Step 18.** Before continuing ahead, let's download the go.mongodb.org/mongo-driver package:

```sh
go get go.mongodb.org/mongo-driver
```

**Step 19.** Let's now work on GetUserById() function. Remember that GetUserById() is for the users to access there own information, Admins can access all the users data, users can only access theirs.

```go
func GetUserById() gin.HandlerFunc{
	return func(c *gin.Context){
		userId := c.Param("user_id") // we are taking the user_id given by the user in json
		// with the help of gin.context we can access the json data send by postman or curl or user
		
		if err := helper.MatchUserTypeToUserId(c, userId); err != nil{
			//checking if the user in admin or not.
			// we will create that func in helper package.
				c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
				return 
			}
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		var user models.User
		
		err := userCollection.FindOne(ctx, bson.M{"user_id": userId}).Decode(&user)
		defer cancel()
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		// if everything goes ok, pass the data of the user (UserModel.go)
		c.JSON(http.StatusOK, user)
	}
}
```

- Taking the user_id from the request and storing it in userId variable.

- creating a variable called user of type User. then simply searching the user in our database with the help of user_id, if the user_id matches, we will store that persons information in user variable.

- If everything goes fine, StatusOk

- we are also handling the errors at every step.

**Step 20.**  Let's now work on GetUsers() function. Remember that only the admin can access the GetUsers() function because it would include the data of all the users. Create a GetUsers() function in the same file as GetUserById(), Login() and SignUp() function:

```go
func GetUsers() gin.HandlerFunc{
	return func(c *gin.Context){
		if err := helper.CheckUserType(c, "ADMIN"); err != nil{
			c.JSON(http.StatusBadRequest, gin.H{"error":err.Error()})
			return
		}
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		//  setting how many records you want per page.
		// we are taking the recodePerPage from c and converting it to int
		recordPerPage, err := strconv.Atoi(c.Query("recordPerPage"))
		// if error or recordPerPage is less than 1, by default we will have 9 records per page
		if recordPerPage<1||err != nil {
			recordPerPage = 9
		}
		// this is just like page number
		page, err1 := strconv.Atoi(c.Query("page"))
		// we want to start with the page number 1 by default.
		if err1 !=nil || page < 1{
			page = 1
		}
		startIndex := (page - 1) * recordPerPage
		startIndex, err = strconv.Atoi(c.Query("startIndex"))
		matchStage := bson.D{{"$match", bson.D{{}},
	}}
		// group all the data based on id, and then count them using $sum. then
		// pushing all the data to the root.
		groupStage := bson.D{{"$group", bson.D{{"_id", bson.D{{"_id", "null"}}},
		{"total_count", bson.D{{"$sum", 1}}},
		{"data", bson.D{{"$push", "$$ROOT"}}},
	}}}
		// in project stage we deside which data should go to the user and which not.
		projectStage := bson.D{
			{"$project", bson.D{
				{"_id", 0},
				{"total_count", 1},
				{"user_items", bson.D{{"$slice", []interface{}{"$data", startIndex, recordPerPage}}}},
			}},
		}
		result, err := userCollection.Aggregate(ctx, mongo.Pipeline{
			matchStage, groupStage, projectStage})
		defer cancel()
		if err != nil{
			c.JSON(http.StatusInternalServerError, gin.H{"error":"error occured while listing user items"})
		}
		// creating a slice called allUser and giving the result value
		var allUsers []bson.M
		if err := result.All(ctx, &allUsers); err != nil {
			log.Fatal(err)
		} 
		c.JSON(http.StatusOK, allUsers[0])
	}
}
```

- Firstly, we will check if the request is coming from the admin or not, we do that by the help of CheckUserType() we created in previous steps.

- Then we are setting how many records you want per page.

- We can do that by taking the recodePerPage from the request and converting it to int, this is done by srtconv.

- If any error occurs in setting recordPerPage or recordPerPage is less than 1, by default we will have 9 records per page

- Similarly we are taking the page number in variable 'page'.

- By default we will have page number as 1 and 9 recordPerPage.

- Then we created three stages (matchStage, groupStage, projectStage). 

- Then we are setting these three stages in our Mongo pipeline using Aggregate() function

- Also we are handling errors are every step.

**Step 21.** Our 'userController.go' is now ready, this is how the 'userController.go' file looks like after completion:

```go
package controllers
import (
	"context"
	"fmt"
	"log"
	"net/http"
	"strconv"
	"time"
	database "github.com/1shubham7/jwt/database"
	helper "github.com/1shubham7/jwt/helpers"
	models "github.com/1shubham7/jwt/models"
	"github.com/gin-gonic/gin"
	"github.com/go-playground/validator/v10"
	"golang.org/x/crypto/bcrypt"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo"
)
var userCollection *mongo.Collection = database.OpenCollection(database.Client, "user")
var validate = validator.New()
func HashPassword(password string) string {
	hashed, err:=bcrypt.GenerateFromPassword([]byte(password), 14)
	if err!=nil{
		log.Panic(err)
	}
	return string(hashed)
}
func VerifyPassword(userPassword, providedPassword string) (bool, string) {
	
	err := bcrypt.CompareHashAndPassword([]byte(providedPassword), []byte(userPassword))
	check := true
	msg := ""
	if err != nil {
		check = false
		msg = fmt.Sprintf("email or password is incorrect.")
	}
	return check, msg
}
func SignUp()gin.HandlerFunc{
	return func(c *gin.Context){
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		var user models.User
		if err := c.BindJSON(&user); err!=nil{
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		validationErr := validate.Struct(user)
		// this is used to validate, but what? see the User struct, and see those validate struct fields
		if validationErr != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": validationErr.Error()})
			return
		}
		// we are using count to help us validate. if you find documents with the user email already
		// then count would be more than 0, and we can then handle that err
		count, err := userCollection.CountDocuments(ctx, bson.M{"email":user.Email})
		defer cancel()
		if err != nil {
			log.Panic(err)
			c.JSON(http.StatusInternalServerError, gin.H{"error":"error occured while checking for the email"})
		}
		password := HashPassword(*user.Password)
		user.Password = &password
		count, err = userCollection.CountDocuments(ctx, bson.M{"phone":user.Phone})
		defer cancel()
		if err != nil {
			log.Panic(err)
			c.JSON(http.StatusInternalServerError, gin.H{"error":"error occured while checking for the phone number"})
		}
		if count > 0 {
			c.JSON(http.StatusInternalServerError, gin.H{"error":"this email or phone number already exists"})
		}
		// by "c.BindJSON(&user)" user already have the information from the website user
		user.Created_at, _  = time.Parse(time.RFC3339, time.Now().Format(time.RFC3339))
		user.Updated_at, _  = time.Parse(time.RFC3339, time.Now().Format(time.RFC3339))
		user.ID = primitive.NewObjectID()
		user.User_id = user.ID.Hex()
		token, refreshToken, _ := helper.GenerateAllTokens(*user.Email, *user.First_name, *user.Last_name, *user.User_type, *&user.User_id)
		
		// giving value that we generated to user
		user.Token = &token
		user.Refresh_token = &refreshToken
		// now let's insert it to the database
		resultInsertionNumber, insertErr :=  userCollection.InsertOne(ctx, user)
		if insertErr != nil {
			msg := fmt.Sprintf("User item was not created")
			c.JSON(http.StatusInternalServerError, gin.H{"error": msg})
			return
		}
		defer cancel()
		c.JSON(http.StatusOK, resultInsertionNumber)
	}
}
func Login() gin.HandlerFunc{
	return func(c *gin.Context) {
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		var user models.User
		var foundUser models.User
		// giving the user data to user variable
		if err := c.BindJSON(&user); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		// finding the user through email and if found, storing it in foundUser variable
		err := userCollection.FindOne(ctx, bson.M{"email": user.Email}).Decode(&foundUser)
		defer cancel()
		if err!=nil{
			c.JSON(http.StatusInternalServerError, gin.H{"error": "email or password is incorrect"})
			return 
		}
		// we need pointer to acess the origina user and foundUser,
		// if we only pass user and foundUser, it will create a new instance of user and foundUser
		isPasswordValid, msg := VerifyPassword(*user.Password, *foundUser.Password)
		defer cancel()
		if isPasswordValid != true{
			c.JSON(http.StatusInternalServerError, gin.H{"error": msg})
			return
		}
		if foundUser.Email == nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": "user not found"})
		}
		token, refreshToken, _ := helper.GenerateAllTokens(*foundUser.Email, *foundUser.First_name, *foundUser.Last_name, *foundUser.User_type, foundUser.User_id)
		helper.UpdateAllTokens(token, refreshToken, foundUser.User_id)
		err = userCollection.FindOne(ctx, bson.M{"user_id":foundUser.User_id}).Decode(&foundUser)
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusOK, foundUser)
	}
}
// GetUsers can only be accessed by the admin.
func GetUsers() gin.HandlerFunc{
	return func(c *gin.Context){
		if err := helper.CheckUserType(c, "ADMIN"); err != nil{
			c.JSON(http.StatusBadRequest, gin.H{"error":err.Error()})
			return
		}
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		//  setting how many records you want per page.
		// we are taking the recodePerPage from c and converting it to int
		recordPerPage, err := strconv.Atoi(c.Query("recordPerPage"))
		// if error or recordPerPage is less than 1, by default we will have 9 records per page
		if recordPerPage<1||err != nil {
			recordPerPage = 9
		}
		// this is just like page number
		page, err1 := strconv.Atoi(c.Query("page"))
		// we want to start with the page number 1 by default.
		if err1 !=nil || page < 1{
			page = 1
		}
		startIndex := (page - 1) * recordPerPage
		startIndex, err = strconv.Atoi(c.Query("startIndex"))
		matchStage := bson.D{{"$match", bson.D{{}},
	}}
		// group all the data based on id, and then count them using $sum. then
		// pushing all the data to the root.
		groupStage := bson.D{{"$group", bson.D{{"_id", bson.D{{"_id", "null"}}},
		{"total_count", bson.D{{"$sum", 1}}},
		{"data", bson.D{{"$push", "$$ROOT"}}},
	}}}
		// in project stage we deside which data should go to the user and which not.
		projectStage := bson.D{
			{"$project", bson.D{
				{"_id", 0},
				{"total_count", 1},
				{"user_items", bson.D{{"$slice", []interface{}{"$data", startIndex, recordPerPage}}}},
			}},
		}
		result, err := userCollection.Aggregate(ctx, mongo.Pipeline{
			matchStage, groupStage, projectStage})
		defer cancel()
		if err != nil{
			c.JSON(http.StatusInternalServerError, gin.H{"error":"error occured while listing user items"})
		}
		// creating a slice called allUser and giving the result value
		var allUsers []bson.M
		if err := result.All(ctx, &allUsers); err != nil {
			log.Fatal(err)
		} 
		c.JSON(http.StatusOK, allUsers[0])
	}
}
func GetUserById() gin.HandlerFunc{
	return func(c *gin.Context){
		userId := c.Param("user_id") // we are taking the user_id given by the user in json
		// with the help of gin.context we can access the json data send by postman or curl or user
		
		if err := helper.MatchUserTypeToUserId(c, userId); err != nil{
			//checking if the user in admin or not.
			// we will create that func in helper package.
				c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
				return 
			}
		var ctx, cancel = context.WithTimeout(context.Background(), 100*time.Second)
		var user models.User
		
		err := userCollection.FindOne(ctx, bson.M{"user_id": userId}).Decode(&user)
		defer cancel()
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		// if everything goes ok, pass the data of the user (UserModel.go)
		c.JSON(http.StatusOK, user)
	}
}
```

**Step 22.** Now we can work on the authentication part. For that we will create a authenticate middleware. Create a folder called 'middleware' and create a file inside it called 'authMiddleware.go'. Enter the following code in the file:

```go
package middleware
import (
	"net/http"
	"github.com/gin-gonic/gin"
	"fmt"
	helpers "github.com/1shubham7/jwt/helpers"
)
func Authenticate() gin.HandlerFunc{
	return func(c *gin.Context) {
		clientToken := c.Request.Header.Get("token")
		if clientToken == "" {
			c.JSON(http.StatusInternalServerError, gin.H{"error":fmt.Sprintf("No Authorization header provided")})
			c.Abort()
			return
		}
		claims, err := helpers.ValidateToken(clientToken)
		if err !="" {
			c.JSON(http.StatusInternalServerError, gin.H{"error":err})
			c.Abort()
			return
		}
		c.Set("email", claims.Email)
		c.Set("first_name", claims.First_name)
		c.Set("last_name", claims.Last_name)
		c.Set("uid", claims.Uid)
		c.Set("user_type", claims.User_type)
		c.Next()
	}
}
```

**Step 23.** Now let's create ValidateToken() function, we will create this function in the 'tokenHelper.go':

```go
func ValidateToken(signedToken string) (claims *SignedDetails, msg string) {
	// this function is basically returning the token
	token, err := jwt.ParseWithClaims(
		signedToken,
		&SignedDetails{},
		func(token *jwt.Token)(interface{}, error){
			return []byte(SECRET_KEY), nil
		},
	)
	if err != nil {
		msg = err.Error()
		return
	}
	// checking if the token is correct or not
	claims, ok := token.Claims.(*SignedDetails)
	if !ok {
		msg = fmt.Sprintf("the token is invalid")
		msg = err.Error()
		return 
	}
	// if the token is expired, give error message
	if claims.ExpiresAt < time.Now().Local().Unix(){
		msg = fmt.Sprintf("token has been expired")
		msg = err.Error()
		return
	}
	return claims, msg
}
```

- ValidateToken takes signedToken and returns SignedDetails of that along with an error message. "" if there is no error.

- ParseWithClaims() function is being used to get us the token and store it in a variable called token.

- Then we are checking if the token is correct or not using the Claims method on token. And we are storing the result in claims variable.

- Then we are checking if the token has expired using ExpiresAt() function, if current time is greater than the ExpiresAt time, it would have expired.

- And then simply return the claims variable as well as the message.

**Step 24.** We are mostly done now, let's do 'go mod tidy', this command checks in your go.mod file, it deletes all the packages/dependencies that we installed but are not using and downloads any dependencies that we are using but haven't downloaded yet.

```sh
go mod tidy
```

![Output](/blog/jwt/three.png)

## Output

With that our JWT authentication project is ready, to finally run the application enter the following command in the terminal:

```sh
go run main.go
```

You will get a similar output:

![Output](/blog/jwt/two.png)

This will get the server up and running, and you can use curl or Postman API for sending request and receiving response. Or you can simply integrate this API with an frontend framework. And with that, our authentication API is ready, pat yourself on the back! 

## Conclusion

In this article we discussed one of the fastest ways to create JWT authentication, we used Gin-Gonic framework for our project. This is not your "just another authentication API". Gin is 300% faster than NodeJS, that makes this authentication API really fast and efficient. The project structure we are using is also an industry level project structure. You can make further changes like storing the SECRET_KEY in the .env file and a lot more to make this API better. You can also find the source code for this project here - [**1Shubham7/go-jwt-token**](https://github.com/1Shubham7/go-jwt-token).

Make sure to follow all the steps in order to create the project and add some more functionality and just play with the code to understand it better. The best way to learn authentication is to create your own projects.