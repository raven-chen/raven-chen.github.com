---
layout: post
title: "Go 的集成测试"
description: "使用 agouti 进行 Go 的浏览器测试与通过 semaphoreCI 搭建 CI"
category:
tags: [go test integration agouti]
---


最近从 Ruby 转到 Go. 新项目 [QOR](https://github.com/qor/qor/admin) 需要浏览器集成测试，一番搜索后发现了 [agouti](agouti.org), 试用一下发现基本算是 Go 版本的 Capybara,正好适合当下的任务. 几天的时间写好了测试并把 CI 跑了起来,这里总结一下经验,希望能对大家有所帮助.

## 浏览器集成测试

agouti 官网上的例子推荐使用 agouti + Ginkgo + Gomega 的组合，本着用的工具越简单，工具本身带来 bug 机率越小的原则, 试验了一下, 最后选择了 agouti + Gomega 的组合, 主要是看中了 Gomega 提供的 one line assertion, 而 Ginkgo 的描述式 DSL 就没什么吸引力了。

### 1. 选择 Driver

最开始习惯性用了 selenium 配合 chrome, 本地很快跑起来，但是在测试写完后设置 CI 时耗费了很多时间在调试 Xvfb 上面, 总是出问题，想着 PhantomJS 不依赖屏幕输出，切换过去，结果与浏览器本身操作相关的例如 `ConfirmPopup` 之类的测试都挂了，这个是 agouti 提供的用以确认 confirm box 的方法，后面联系 agouti 作者请求帮助, 他建议如果 CI 有可以运行的 chromedriver 那用这个 driver 会比较好. 切换之后, CI 可以正常工作了.

所以我的经验是 **确保你的 PATH 中有可执行的 chromedriver. 然后使用 ChromeDriver**

具体的例子可以在[这里](https://github.com/qor/qor/blob/master/test/integration)查看

### 2. 搭建环境

Go 1.4 引入了 TestMain 方法, 使得测试的开始设置和收尾非常方便, 配合 agouti 的实现如下, 代码在[这里](https://github.com/qor/qor/blob/master/test/integration/setup_integration_test.go)

    var (
        baseUrl = fmt.Sprintf("http://localhost:%v/admin", PORT)
        driver  *agouti.WebDriver
        page    *agouti.Page
    )

    func TestMain(m *testing.M) {
        var t *testing.T
        var err error

        driver = agouti.ChromeDriver() // 设置 driver
        driver.Start()

        go Start(PORT) // 启动你要测试的程序

        page, err = driver.NewPage() // 初始化页面对象
        if err != nil {
            t.Error("Failed to open page.")
        }

        RegisterTestingT(t) // 注册 Gomega
        test := m.Run() // 启动测试

        driver.Stop() // 关闭 driver
        os.Exit(test) // 结束测试
    }

    // 测试出错时打印堆栈信息并关闭 driver, 需要在每个测试开始时用 defer 调用
    func StopDriverOnPanic() {
        var t *testing.T
        if r := recover(); r != nil {
            debug.PrintStack()
            fmt.Println("Recovered in f", r)
            driver.Stop()
            t.Fail()
        }
    }

设置好后我们写个简单的测试来验证一下环境是否搭建成功

    func TestPage(t *testing.T) {
        defer StopDriverOnPanic()

        Expect(page.Navigate("localhost:3000").To(Succeed())
    }

如果环境没问题,你应该可以看到 chrome 启动并且访问了你所指定的地址.

### 3. 实现浏览器测试

在上一步的设置的基础上, 可以来实现我们的测试了. 基本是以 css selector 来模拟操作, 在实现的过程中 有几点需要注意的地方

- 尽量使用有唯一性的 css selector. 确保可以精准定位到你所期望操作的元素.
- 在执行断言前, 最好使用 `Eventually(page).Should(...)` 来确保之前的操作已经完成,特别是在提交表单之后,有时虽然在本地可以正常断言,但是一般 CI 的性能都不如我们的开发机器导致本地通过 CI 红色的情况.
- 尽量使用 webdriver 底层协议的方法来操作浏览器, 例如 [AcceptAlert](https://code.google.com/p/selenium/wiki/JsonWireProtocol#/session/:sessionId/accept_alert), 出错的几率小很多

下面是在项目中实现的一个测试 form 的例子, 源码在[这里](https://github.com/qor/qor/blob/master/test/integration/form_test.go)

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
        page.Find("#QorResourceGender_chosen .chosen-drop ul.chosen-results li[data-option-array-index='1']").Click()

        // Select many
        page.Find("#QorResourceLanguages_chosen .search-field input").Click()
        page.Find("#QorResourceLanguages_chosen .chosen-drop ul.chosen-results li[data-option-array-index='0']").Click()

        page.Find("#QorResourceLanguages_chosen").Click()
        Expect(page.Find("#QorResourceLanguages_chosen .chosen-drop ul.chosen-results li[data-option-array-index='1']").Click()).To(Succeed())

        // File upload
        Expect(page.Find("input[name='QorResource.Avatar']").UploadFile("fixtures/ThePlant.png")).To(Succeed())

        page.FindByButton("Save").Click()
    }


## 搭建 CI

最开始同事推荐使用 Drone, 因为这个工具是用 Go 写的，试用了一下发现只有一个文本框提供命令输入，调试实在太费力，后来发现了 [semaphore CI](semaphoreci.com). 各项功能都比较符合需要 推荐使用的理由有这几个

1. 对开源项目免费
2. 支持 Launch SSH 功能，测试失败后，你可以请求一个限时一个小时的 ssh 许可，自己登陆到 CI 上去调试，这个在搭建环境时非常方便，相信以后 CI 出了问题，用这个功能也会有很大帮助.
3. 测试环境支持比较完善, [Supported stacks](https://semaphoreci.com/docs/supported-stack.html) 从这里可以看到，常用的语言和库都已经安装好了，这次使用的 chromedriver 和 Xvfb 就是都默认支持，无需自己配置，很便捷.
4. 通知方式全面，邮件通知，基础的 github, bitbucket 的 webhook,campfire, slack 的集成都支持，便于开发时接收 CI 结果.
5. 支持并行测试，配置好命令后它会把每条命令都生成一条记录，你可以选择这个记录属于哪个命令队列, 这是我自己的配置

    mysql -uroot -psemaphoredb -e "CREATE DATABASE IF NOT EXISTS "qor_test" CHARACTER SET utf8 COLLATE utf8_general_ci;" // 公共任务
    go get ./... // 公共任务

    cd admin // 队列1
    TEST_ENV=CI DB_USER=root DB_PWD=semaphoredb go test // 队列1

    cd test/integration // 队列2
    TEST_ENV=CI DB_USER=root DB_PWD=semaphoredb go test // 队列2

这个例子里，准备数据库和包安装都设置在 Setup 队列里, admin 目录下是单元测试，放进 Thread#1 队列，test/integration 则用 Thread#2 队列，这样就可以同时跑单元和集成测试了。这只是一个简单地对于并行测试的应用, 相信这个功能以后可以有更多的用处.

### 设置 semaphoreCI
和大多数 CI 托管项目一样, 用你的 github/bitbucket 项目登陆并给予 semaphoreCI 权限, 接着选择你所要测试的项目, 在 build setting 里选择 Go 的版本并设置跑起项目所需的命令, 然后可以手动运行测试了. 有新的 commit 或 pull request 提交时会自动触发测试.

到这里 pull request 上的绿色小勾就出现了，睡觉也安稳了 :p
