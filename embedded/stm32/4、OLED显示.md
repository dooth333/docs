## 课程中的OLED驱动函数

### 区域划分：

被分成4*16：

![image-20220804105801428](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220804105801428.png)

### 显示函数：

| **函数**                               | **作用**             |
| -------------------------------------- | -------------------- |
| OLED_Init();                           | 初始化               |
| OLED_Clear();                          | 清屏                 |
| OLED_ShowChar(1, 1, 'A');              | 显示一个字符         |
| OLED_ShowString(1, 3,  "HelloWorld!"); | 显示字符串           |
| OLED_ShowNum(2, 1, 12345, 5);          | 显示十进制数字       |
| OLED_ShowSignedNum(2, 7, -66, 2);      | 显示有符号十进制数字 |
| OLED_ShowHexNum(3, 1, 0xAA55, 4);      | 显示十六进制数字     |
| OLED_ShowBinNum(4, 1, 0xAA55, 16);     | 显示二进制数字       |

### 注意事项：

1. 调用时要根据不同接线方法去OLED.c中配置SCL和SDA的引脚，并且初始化函数也要更改一下。