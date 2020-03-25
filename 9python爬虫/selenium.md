### `selenium

https://www.bilibili.com/video/av64421994

#### 1.简介

Selenium is an umbrella project for a range of tools and libraries that enable and support the automation of web browsers.

#### 2.简单使用

1. 安装selenium模块

   pip install selenium

2. 下载浏览器驱动

   http://chromedriver.storage.googleapis.com/index.html

3. 使用python中的selenium操作浏览器

   ```python
   from selenium import webdriver
   
   # 初始化谷歌浏览区驱动对象
   chromedriver = webdriver.Chrome(r'D:\drivers\chromedriver.exe')
   # 使用驱动访问百度
   chromedriver.get("http://www.baidu.com")
   # 选择网页上的元素
   element = chromedriver.find_element_by_id('kw')
   # 在该元素上输入hello，并回车搜索
   element.send_keys('hello\n')
   ```

#### 3.选择元素的基本方法

- 根据id获取元素

  ```
  find_element_by_id
  ```

- 根据class获取元素

  ```
  find_elements_by_class_name
  ```

- 根据标签名获取元素

  ```
  chromedriver.find_element_by_tag_name('div')
  ```

- 设置最大等待时间

  ```
  chromedriver.implicitly_wait(10)
  ```

#### 5.操作元素的方法

- 点击元素

  click()

- 输入内容
  
  senk_keys()
  
- 获取内容

  text

- 获取元素属性

  get_attribute()

#### 6.CSS选择器

- 选择子节点和后代节点

  ```css
  #aa > #bb
  #aa #bb
  ```

- 根据属性选择节点

  ```css
  div#aa
  div[name='ball']
  ```

- “与”关系

  ```css
p,span
  ```
  
- 根据次序选择节点

  - 父元素的第几个子节点

      ```css
      span:nth-child(2)
      span:nth-last-child(1)
      ```

  - 父元素第几个某类型的子节点
  
    ```css
    span:nth-of-type(2)
    span:nth-last-of-type(2)
    ```
  
  - 父元素的奇偶数子节点
  
    ```css
    span:nth-child(even)
    span:nth-child(odd)
    ```
  
- 兄弟节点

  ```css
  /* 下面一个兄弟节点 */
  p + span
  /* 下面所有某类型的结点 */
  p ~ span
  ```


#### 7.frame/窗口 切换

```python
switch_to.frame("frame1")
# 切换回原来的上下文
switch_to.default_content()
```

```python
handle_list = chromedriver.window_handles
chromedriver.switch_to.window(handle)
# 获取当前handle
chromedriver.current_window_handle
```

#### 8.选择框

- radio

- checkbox

- select

  1. select单选框

      ```python
      from selenium.webdriver.support.ui import Select

      select = Select(chromedriver.find_element_by_id("select_id"))

      select.select_by_visible_text("篮球")
      ```
  
  2. select多选框
  
     ```python
     from selenium.webdriver.support.ui import Select
     
     select = Select(chromedriver.find_element_by_id("select_id"))
     
     #清除所有已默认选择的
     select.deselect_all()
     
     select.select_by_visible_text("篮球")
     select.select_by_visible_text("足球")
     ```

#### 9.鼠标操作

单击、双击、左键、右键、移动、拖拽等

```python
from selenium.webdriver.common.action_chains import ActionChains

ac = ActionChains(chromedriver)
#将鼠标移动到某个元素上
ac.move_to_element(
    chromedriver.find_element_by_id("pic")
).perform()
```

#### 10.弹出框

- alert

  ```python
  chromedriver.switch_to.alert.text
  chromedriver.switch_to.alert.accept()
  ```

- comfirm

  ```python
  chromedriver.switch_to.alert.text
  chromedriver.switch_to.alert.accept()
  chromedriver.switch_to.alert.dismiss()
  ```

- promat

#### 11.xpath 选择页面元素

| 符号                  | 符号解释                       | 示例                       | 示例解释                               |
| --------------------- | ------------------------------ | -------------------------- | -------------------------------------- |
| /                     | 根节点                         | /html                      |                                        |
| //                    | 相对路径                       | //div                      | 选择页面中所有div标签                  |
| [@id="he"]            | 根据元素属性选择               | //*[@id="he"]              | 选择所有id为he的标签                   |
| contains(@id,"he")    | 包含                           | //*[contains(@id,"he")]    | 选择所有id包含he的标签                 |
| starts-with(@id,"he") | 以...开头                      | //*[starts-with(@id,"he")] | 选择所有id以he开头的标签               |
| ends-with(@id,"he")   | 以...结尾                      | //*[ends-with(@id,"he")]   | 选择所有id以he结尾的标签               |
| td[2]                 | 选择某类型的第N个              | //table/td[2]              | 选择table标签下td标签的第二个          |
| td[last()]            | 选择某类型的最后一个           | //table/td[last()-1]       | 选择table标签下td标签的倒数第二个      |
| td[position()>3]      | 选择某个范围                   | //table/td[position()>3]   | 选择table标签下td标签的前两个          |
| \|                    | 组选择                         | //p \| //h3                | 选择所有的p标签和h3标签                |
| ..                    | 选择父节点                     | //div[id='he']/..          | 选择id为he的div标签的父节点            |
| follow-sibling::p     | 选择后续兄弟节点所有某类型节点 | //div/follow-sibling::p    | 选择div标签的后续兄弟节点中的所有p标签 |
| preceding-sibling::p  | 选择前面兄弟节点所有某类型节点 | //div/preceding-sibling::p | 选择div标签的前面兄弟节点中的所有p标签 |

