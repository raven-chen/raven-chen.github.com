# How to Write Browser Testing with Go

### Introduction

This tutorial guides you to test your web application in browser with [Agouti](http://agouti.org).

Agouti is an acceptance or integration testing library written in Go. It the only but a good testing library since Go community is pretty new.

In the tutorial, we assume you are working on a Unix-like operating system.

### Prerequisites

You need [Go 1.4+](http://golang.org/doc/install), [sqlite](https://www.sqlite.org/download.html) and [chromedriver](https://sites.google.com/a/chromium.org/chromedriver/getting-started) installed.

## Step 1, Build an Application for Test
In this section, we will create an application based on [QOR](https://github.com/qor/qor) for our test.

[QOR](https://github.com/qor/qor) is a set of libraries written in Go that abstracts common features needed for business applications, CMSs, and E-commerce systems. You don't need to necessarily understand how QOR works, we just need a simple web application for our tutorial.

Create a new folder called `go_integration_test` under your go working directory

``` sh
mkdir go_integration_test
cd go_integration_test
```

Then create a `main.go` file.

``` sh
touch main.go
```

Copy & paste below code into it.

``` go
package main

import (
	"fmt"
	"net/http"

	"github.com/jinzhu/gorm"
	_ "github.com/mattn/go-sqlite3"
	"github.com/qor/qor"
	"github.com/qor/qor/admin"
)

type User struct {
	gorm.Model
	Name   string
	Gender string
}

var (
	DB gorm.DB
)

func main() {
	Start(9000)
}

func AdminConfig() (mux *http.ServeMux) {
	DB, _ = gorm.Open("sqlite3", "demo.db")
	DB.AutoMigrate(&User{})

	Admin := admin.New(&qor.Config{DB: &DB})
	user := Admin.AddResource(&User{}, &admin.Config{Menu: []string{"User Management"}})
	user.Meta(&admin.Meta{Name: "Gender", Type: "select_one", Collection: []string{"Male", "Female"}})

	mux = http.NewServeMux()
	Admin.MountTo("/admin", mux)

	return
}

func Start(port int) {
	mux := AdminConfig()
	http.ListenAndServe(fmt.Sprintf(":%v", port), mux)
}
```

Now, install all dependencies by

``` sh
go get ./...
```

Then type

``` sh
go run main.go
```

You should see [QOR](https://github.com/qor/qor) is running at `http://localhost:9000/admin`.

## Step 2, Configure Test Environment
In this section we will configure our test environment to run test against our application.

Create a test file in `go_integration_test` directory first

``` sh
touch main_test.go
```

Define package and import packages we need. Follow Go's convention, test and program should in the same package.

``` go
package main

import (
	"fmt"
	"os"

	"testing"

	. "github.com/onsi/gomega" // . means this package's method can be called without package prefix
	"github.com/sclevine/agouti"
)
```

Define variables we will use in test

``` go
const (
	PORT = 4444
)

var (
	baseUrl = fmt.Sprintf("http://localhost:%v/admin", PORT)
	driver  *agouti.WebDriver
	page    *agouti.Page
)
```

Setup Agouti, Go 1.4 introduced the `TestMain` function, making test setup and teardown much easier. so we setup agouti inside it. **please ensure you have an executable chromedriver in your PATH** [check here for chromedriver installation](https://sites.google.com/a/chromium.org/chromedriver/getting-started)

``` go
func TestMain(m *testing.M) {
	var t *testing.T
	var err error

	driver = agouti.ChromeDriver() // choose browser driver
	driver.Start()

	go Start(PORT) // start our program

	page, err = driver.NewPage() // get page object from driver, this is what we will use to perform browser testing
	if err != nil {
		t.Error("Failed to open page.")
	}

	RegisterTestingT(t)
	test := m.Run() // start test

	driver.Stop() // close driver after test
	os.Exit(test)
}
```

Let's write a simple test in `main_test.go` to see if our environment is ok.

``` go
func TestEnv(t *testing.T) {
	Expect(page.Navigate(fmt.Sprintf("%v/users", baseUrl))).To(Succeed())
}
```

Install test packages by

``` sh
go get -t ./...
```

Now, run

``` sh
go test
```

If your environment is ok, you should see chrome showed up and terminal shows PASS like this

```
2015/07/11 09:01:13 Start [GET] /admin/user
2015/07/11 09:01:13 Finish [GET] /admin/user Took 10.89ms
PASS
ok  	github.com/raven-chen/go_integration_test	2.674s
```

## Step 3, Test Our Application
In this section, we will start write test based on previous 2 steps, At the beginning of this article, we defined a resource user(check [here](https://github.com/qor/qor) for more information about QOR). Let's try test create an user.

Create a test called `TestCreateUser`, please check comment in code to see what the code does.

``` go
func TestCreateUser(t *testing.T) {
	var user User
	userName := "user name"

	Expect(page.Navigate(fmt.Sprintf("%v/users", baseUrl))).To(Succeed()) // visit user page
	Expect(page.Find(".qor-button--new").Click()).To(Succeed())                     // click add user button

	page.Find("input[name='QorResource.Name']").Fill(userName) // fill in user name

	page.FindByButton("Add User").Click() // submit form

	DB.Last(&user) // query the user we just created

	if user.Name != userName { // assert it created as we expected
		t.Error("user name not set")
	}
}
```

Then run

``` sh
go test
```

You should see chrome acts like a real man is using your program in it. your terminal should display PASS same as our smoke test.

Let's try a more complicate case. User has `Gender` attribute which is a select one widget in the form. Expand our test to select `Gender` for user. the test becomes this

``` go
func TestCreateUser(t *testing.T) {
	var user User
	userName := "user name"

	Expect(page.Navigate(fmt.Sprintf("%v/users", baseUrl))).To(Succeed()) // visit user page
	Expect(page.Find(".qor-button--new").Click()).To(Succeed())           // click add user button

	page.Find("input[name='QorResource.Name']").Fill(userName) // fill in user name

	// Select Gender
	page.Find("#User_0_Gender_chosen").Click()
	page.Find("#User_0_Gender_chosen .chosen-drop ul.chosen-results li[data-option-array-index='0']").Click()

	page.FindByButton("Add User").Click() // submit form

	DB.Last(&user) // query the user we just created

	if user.Name != userName { // assert it created as we expected
		t.Error("user name not set")
	}

	if user.Gender != "Male" {
		t.Error("user gender not set")
	}
}
```

we added

``` go
	// Select Gender
	page.Find("#User_0_Gender_chosen").Click()
	page.Find("#User_0_Gender_chosen .chosen-drop ul.chosen-results li[data-option-array-index='0']").Click()
```

to the test, these code simulates user to operate the form by css selector. You can visit `http://localhost:9000/admin/user/new` to see the how these css selector works.


That's it ! Your first browser integration test has been created successfully !

## Common usage of agouti
This is sample test that covers most of operations in form. You can get full list of supported functions in [agouti api](http://godoc.org/github.com/sclevine/agouti/api).

Please note that this test is not based on our sample application, It is a QOR's test, You can find how to run this test [here](https://github.com/qor/qor/blob/master/test/integration).

``` go
func TestForm(t *testing.T) {
	SetupDb(true)
	defer StopDriverOnPanic()

	var user User
	var languages []Language
	userName := "user name"
	address := "an address"

	langEN := &Language{Name: "en"}
	langCN := &Language{Name: "cn"}
	DB.Create(&langEN)
	DB.Create(&langCN)

	Expect(page.Navigate(fmt.Sprintf("%v/user", baseUrl))).To(Succeed())
	Expect(page.Find("#plus").Click()).To(Succeed())
	Expect(page).To(HaveURL(fmt.Sprintf("%v/user/new", baseUrl)))

	// Text input
	page.Find("#QorResourceName").Fill(userName)

	// Select one
	page.Find("#QorResourceGender_chosen").Click()
	page.Find("#QorResourceGender_chosen .chosen-drop ul.chosen-results li[data-option-array-index='1']").Click()

	// Select many
	page.Find("#QorResourceLanguages_chosen .search-field input").Click()
	page.Find("#QorResourceLanguages_chosen .chosen-drop ul.chosen-results li[data-option-array-index]:nth-child(1)").Click()

	page.Find("#QorResourceLanguages_chosen").Click()
	Expect(page.Find("#QorResourceLanguages_chosen .chosen-drop ul.chosen-results li[data-option-array-index]:nth-child(2)").Click()).To(Succeed())

	// Nested resource
	page.Find("#QorResourceProfileAddress").Fill(address)

	// Rich text
	Expect(page.Find(".redactor-box")).To(BeFound())

	// File upload
	Expect(page.Find("input[name='QorResource.Avatar']").UploadFile("fixtures/ThePlant.png")).To(Succeed())

	page.FindByButton("Save").Click()

	DB.Preload("Profile").Last(&user)
	DB.Model(&user).Related(&languages, "Languages")

	if user.Name != userName {
		t.Error("text input for user name not work")
	}

	if user.Gender != "Male" {
		t.Error("select_one for gender not work")
	}

	if len(languages) != 2 {
		t.Error("select_many for languages not work")
	}

	if user.Profile.Address != address {
		t.Error("nested resource for profile not work")
	}

	avatarFile := fmt.Sprintf("public%v", user.Avatar.Url)
	if _, err := os.Stat(avatarFile); os.IsNotExist(err) {
		t.Error("file uploader for avatar not work")
	} else {
		os.Remove(avatarFile)
		// Remove uploaded .original file
		// File path looks like public/system/users/1/Avatar/ThePlant20150508172715879986152.original.png
		filePaths := strings.Split(avatarFile, ".")
		os.Remove(fmt.Sprintf("%v.%v.original.%v", filePaths[0], filePaths[1], filePaths[2]))
	}
}
```


## References

- The source code of this article could be found [here](https://github.com/raven-chen/go_integration_test)
- The agouti offical site <http://agouti.org/>
- For more usage example <https://github.com/qor/qor/tree/master/test/integration>
