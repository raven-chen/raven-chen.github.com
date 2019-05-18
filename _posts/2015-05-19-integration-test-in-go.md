---
layout: post
title: "Integration test in Go"
description: "Browser test in Go with agouti and setup CI via semaphoreCI"
category:
tags: [go,test,integration,agouti]
---

# Integration testing Go applications with agouti

Agouti is an integration and acceptance testing library for Go. This article shows you how to use it with ChromeDriver for browser testing web applications.

What is the best way to do integration test ? My answer is a indefatigable man that completely understand your application, He can test your application carefully on **real browser** every time you make a change. I know this is "mission impossible". Fortunately, we have another choice -- "Integration test with real browser". You understand your application and computer is indefatigable, So all we need to do is make computer operate browser.

We wanted to add real browser integration testing to our new project [QOR](https://github.com/qor/qor/admin). This is an open source CMS that focus on e-commerce. After investigating several libraries, we decided to use [agouti](http://agouti.org). It’s a bit like a Go version of Capybara. After spending several days with it I managed to get CI setup and several tests working. This blog will introduce our experience with it and I hope you will find it helpful for your own projects.

## Setup integration tests

The Agouti website recommends using a `agouti + Ginkgo + Gomega` combination. But with the reasoning that the simpler your tool is, the less bugs your tool will introduce, we chose `agouti + Gomega` since Gomega provides one line assertions and we didn’t find Ginkgo’s descriptive DSL very attractive.

### 1. Choose a driver

In the beginning, we chose Selenium + Chrome. This combination worked fine in our local environment, but when setting up CI, we ran into problems with Xvfb. We then switched to PhantomJS 2.0 as it doesn't need Xvfb to run. But then we ran into another problem: tests involving browser confirmation boxes/alerts failed. The Agouti author suggested we include an executable chromedriver on CI: Success!

So our experience is **ensure you have an executable chromedriver in your PATH, then use ChromeDriver**.

You can find a real example [here](https://github.com/qor/qor/blob/master/test/integration).

### 2. Setup environment

Go 1.4 introduced the TestMain function, making test setup and teardown much easier. Our implementation is as below and you can find the code [here](https://github.com/qor/qor/blob/master/test/integration/setup_integration_test.go).

    ``` Go
    var (
        baseUrl = fmt.Sprintf("http://localhost:%v/admin", PORT)
        driver  *agouti.WebDriver
        page    *agouti.Page
    )

    func TestMain(m *testing.M) {
        var t *testing.T
        var err error

        driver = agouti.ChromeDriver() // set driver
        driver.Start()

        go Start(PORT) // start your application

        page, err = driver.NewPage() // initialize page object
        if err != nil {
            t.Error("Failed to open page.")
        }

        RegisterTestingT(t) // register Gomega
        test := m.Run() // start test

        driver.Stop() // close driver
        os.Exit(test) // exit test
    }

    // print stack and close driver when test has exception.
    func StopDriverOnPanic() {
        var t *testing.T
        if r := recover(); r != nil {
            debug.PrintStack()
            fmt.Println("Recovered in f", r)
            driver.Stop()
            t.Fail()
        }
    }
    ```


Let’s test if the environment is setup correctly

    ``` Go
    func TestPage(t *testing.T) {
        defer StopDriverOnPanic()

        Expect(page.Navigate("localhost:3000").To(Succeed())
    }
    ```

If the environment is ok, you should see chrome start and access the address you specified in the setup.

### 3. Implement tests

We can start our tests now. The tests basically simulates user operation based on css selectors. There are a few things you should be aware of:

- Try to use unique css selectors to ensure you can locate the element on which you want to operate
- Before running assertions, it is a good idea to use `Eventually(page).Should(…)` to make sure previous operations have been processed, especially after a form has been submitted. Sometimes test work fine locally but fail on CI since most CI systems are not as fast as our local computers.
- Use the webdriver protocol function to operate the browser, like [AcceptAlert](https://code.google.com/p/selenium/wiki/JsonWireProtocol#/session/:sessionId/accept_alert), as it is more stable.

Below is a sample for testing a form. The source code is [here](https://github.com/qor/qor/blob/master/test/integration/form_test.go)

    ``` Go
    func TestForm(t *testing.T) {
        SetupDb(true)
        defer StopDriverOnPanic()

        Expect(page.Navigate(fmt.Sprintf("%v/user", baseUrl))).To(Succeed())
        Expect(page.Find("#plus").Click()).To(Succeed())
        Expect(page).To(HaveURL(fmt.Sprintf("%v/user/new", baseUrl)))

        // Text input
        page.Find("#QorResourceName").Fill(userName)

        // Select one
        page.Find("#QorResourceGender_chosen").Click()
        page.Find("#QorResourceGender_chosen .chosen-drop ul.chosen-results li[data-option-array-index=‘1’]").Click()

        // Select many
        page.Find("#QorResourceLanguages_chosen .search-field input").Click()
        page.Find("#QorResourceLanguages_chosen .chosen-drop ul.chosen-results li[data-option-array-index=‘0’]").Click()

        page.Find("#QorResourceLanguages_chosen").Click()
        Expect(page.Find("#QorResourceLanguages_chosen .chosen-drop ul.chosen-results li[data-option-array-index=‘1’]").Click()).To(Succeed())

        // File upload
        Expect(page.Find("input[name=‘QorResource.Avatar’]").UploadFile("fixtures/ThePlant.png")).To(Succeed())

        page.FindByButton("Save").Click()
    }
    ```


## Setup CI

We started with Drone because it is written in Go, but found it only provides a textarea in which to input commands, which we thought was very inconvenient. So we tried [semaphore CI](semaphoreci.com) and found it perfectly fit our requirements:

1. It is free for open source projects
2. It provides a `Launch SSH` feature, so you can request a 1 hour permission to login to CI to debug your tests after your tests have finished. This is very helpful for CI debugging.
3. [Supports stacks](https://semaphoreci.com/docs/supported-stack.html) save setup time. Popular languages and libraries are installed, like the chromedriver and Xvfb we use locally.
4. Notification support is good including email, GitHub/Bitbucket hooks, Campfire and Slack.
5. It supports parallel tests. Each command line you type can be set to queue. This is our configuration:

    ``` sh
    mysql -uroot -psemaphoredb -e "CREATE DATABASE IF NOT EXISTS "qor_test" CHARACTER SET utf8 COLLATE utf8_general_ci;" // Setup thread
    go get ./… // Setup thread

    cd admin // Thread 1
    TEST_ENV=CI DB_USER=root DB_PWD=semaphoredb go test // Thread 1

    cd test/integration // Thread 2
    TEST_ENV=CI DB_USER=root DB_PWD=semaphoredb go test // Thread 2
    ```

The database setup and package installation are in the Setup thread which is executed before all other threads. Our unit tests are located under /admin in thread#1, test/integration is our integration test in thread#2. Then we can run unit and integration tests at same time. This is just a simple usage of parallel tests. We believe this could be useful to us in the future.

### Setup Semaphore CI

Like most CI systems, login with your GitHub/Bitbucket account and authorize Semaphore. Then select the project you want to test, select the Go version, type your commands, and you’re done! New commits and pull requests will trigger test builds automatically.

Now you should see the green (or red) icons in your pull request page, so you can sleep better (or worse) at night :p


