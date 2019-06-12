---
title: selenium爬虫总结
date: 2019-03-15 17:47:29
tags:
  - selenium
  - python
  - 爬虫
---

对自己在使用selenium对有复杂加密的网站的爬取过程中遇到的问题做总结。

<!-- more -->

## selenium的基本用法

我一般都使用Chrome浏览器

```python
from selenium.webdriver import Chrome, ChromeOptions

chrome_options = ChromeOptions()
driver = Chrome(options=chrome_options, executable_path='./chromedriver')
driver.get('https://www.google.com/')
```

可以调用`chrome_options`的方法来添加各种参数和设置，然后在生成`Chrome`对象的时候传递进去。

`chrome_options`常用的方法:

- `add_extension`(添加扩展)
- `add_argument`(添加参数)
- `set_headless`(设为无头浏览器)

## 等待页面加载完毕

1. `sleep`

   这个就是time模块里的sleep函数，让解释器停止运行一段时间，这样有个弊端就是无论这期间页面加载完毕了没有，都不会停止sleep，相当于浪费时间。

2. `implicitly_wait`

   即隐式等待，这个是driver的方法，比如`driver.implicitly_wait(15)`，就是设置隐式等待时间为15秒，这个设置好之后，在整个driver的生命周期内都起作用。意思相当于设置了一个最长等待时间，等待的是**整个页面**的加载完毕。如果超过了这个时间，还是没有加载好，那就会接着往下运行，那么下面就开有可能会出错。如果没超过这个时间就加载好了，那就直接往下运行，无需再等待。

3. `WebDriverWait`

   即显式等待。可以指定等待**某个元素**的出现比如：

   ```python
   from selenium import webdriver
   from selenium.webdriver.support.wait import WebDriverWait
   from selenium.webdriver.support import expected_conditions as EC
   from selenium.webdriver.common.by import By
   
   driver = webdriver.Chrome()
   driver.implicitly_wait(15)
   driver.get('https://www.google.com/')
   locator = (By.ID, 'gsr')
   
   try:
       ele = WebDriverWait(driver, 20).until(EC.presence_of_element_located(locator))
   finally:
       driver.quit()
   
   ```

   隐性等待和显性等待可以同时用，等待的最长时间取两者之中的大者。

   完整的等待状态列表可以看这里:[module-selenium.webdriver.support.expected_conditions](https://selenium-python-zh.readthedocs.io/en/latest/api.html?highlight=expected%20conditions%20suppor#module-selenium.webdriver.support.expected_conditions)


## 查找元素

driver和web element都可以通过多种方法查找想要的元素，比如有`find_element_by_id`、`find_element_by_css_selector`、`find_element_by_xpath`等等。给方法名里的element加s就可以找所有符合条件的元素。

   **如果遇到有iframe的**

   先`driver.switch_to.frame(iframe)`，然后再查找元素，完毕之后，再调用`driver.switch_to.default_content()`跳出这个iframe。

   ## 如何新开一个标签页

```python
driver.execute_script("window.open('about:blank','_blank');")  # 新开一个空白标签页
driver.switch_to.window(driver.window_handles[len(driver.window_handles) - 1])  # 进入新开的标签页
driver.get('https//www.google.com/')  # 打开链接
# ......
driver.close()  # 关闭当前的标签页
driver.switch_to.window(self.driver.window_handles[len(self.driver.window_handles) - 1])  # 回到前一个标签页，或者其他的，可以任意指定
```

**注意**：`driver.close`和`driver.quit`的不同。

close方法的文档是：Closes the current window."，仅仅关闭当前的窗口。

而quit方法的文档里写的是："Closes the browser and shuts down the ChromeDriver executable that is started when starting the ChromeDriver"，意思就是完全退出了。

## 键盘鼠标模拟操作

```python
from selenium.webdriver import ActionChains
# ......
# element = xxx
action = ActionChains(driver)
action.move_to_element(element).perform()
action.click().perform()
action.send_keys('abcdefg')

action.click_and_hold(swipe_button).perform()  # 点击按住鼠标
action.move_by_offset(580, 0).perform()  # 拖动

```

详细信息见:[module-selenium.webdriver.common.action_chains](https://selenium-python-zh.readthedocs.io/en/latest/api.html?highlight=actionchains#module-selenium.webdriver.common.action_chains)

## 反反爬虫

1. 隐藏`window.navigator.webdriver`

   正常通过driver启动的浏览器，会有一些和正常的浏览器不同的值。在js里调用`window.navigator.webdriver`，对driver启动的返回True，对正常的浏览器返回False。有些网站可能会通过这个值来禁止selenium的爬取。

   解决方法：`chrome_option.add_experimental_option('excludeSwitches', ['enable-automation'])`，这样设置chrome_option以后，启动的浏览器里这个值`window.navigator.webdriver`就和正常浏览器的一样了。

   