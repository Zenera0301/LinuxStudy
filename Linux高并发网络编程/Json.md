# Json

cjson的使用 

\- 压缩包解压缩，直接使用里边的cJSON.c和cJSON.h即可 

\- 链接时还需要加上-lm 表示链接math库 



## json的格式 

#### 1 json数组

- char array[23] = "slkjflajslfd"; 
- 中括号，中括号里面可以是下面这些类型，不一定非得是哪一种类型：[整形, 字符串, 布尔类型, json数组, json对象] 
- [123, 123.2, true, false, [12, 34, 56, "hello, world"]] 



#### 2 json对象

由{}表示，是一些键值对 

- ```c
  { 
      "name":"zhang3",  
      "name2":"li4" 
  } 
  ```

- key值: 必须是 字符串, 不重复 

- value值: 整形, 字符串 , 布尔，json对象, json数组



#### 3 json数组+json对象

```c
{ 
    "name1":"zhang3",  

    "name2":"li4", 

    "张三":{ 

        "别名":"老王", 

        "性别":"男", 

        "年龄":34, 

        "孩子":["小红", "小绿", "小黑"] 
    } 
} 
```





### CJson

C语言json开源解析库 - cjson 

使用，解压包后，将cJSON.c和cJSON.h两个文件拷贝到自己的项目文件下，编译的时候只需要gcc cJSON.c main.c  -o xxx ，然后./xxx 即可

#### 生成json文件

```c
○ 创建一个json对象 

cJSON *cJSON_CreateObject(void); 



○ 往json对象中添加数据成员 

cJSON *object, // json对象 

const char *string, // key值 

cJSON *item // value值（int，string，array，obj） 

void cJSON_AddItemToObject(); 



○ 创建一个整型值 

cJSON *cJSON_CreateNumber(double num); 



○ 创建一个字符串 

cJSON *cJSON_CreateString(const char *string); 



○ 创建一个json数组 

cJSON *cJSON_CreateArray(void); -- 空数组 



○ 创建默认有count个整形值的json数组 

▪ int arry[] = {8,3,4,5,6}; 

▪ cJSON_CreateIntArray(arry, 5); 

cJSON *cJSON_CreateIntArray(const int *numbers,int count); 



○ 往json数组中添加数据成员 

void cJSON_AddItemToArray(cJSON *array, cJSON *item); 



○ 释放jSON结构指针 

void cJSON_Delete(cJSON *c) 



○ 将JSON结构转化为字符串 

▪ 返回值需要使用free释放 

▪ FILE* fp = fopen(); 

▪ fwrite(); 

▪ fclose(); 

char *cJSON_Print(cJSON *item);

```



实现下面的json数据格式：

```json
{
	"奔驰":	{
		"factory":	"一汽大众",
		"last":	31,
		"price":	83,
		"sell":	49,
		"sum":	80,
		"other":[124, true, "hello, world", {"梅赛德斯奔驰":"心所向, 持以恒"}]
	}
}
```



```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include "cJSON.h"

int main(int argc, const char* argv[])
{
    // 创建对象
    cJSON* obj = cJSON_CreateObject();

    // 创建子对象
    cJSON* subObj = cJSON_CreateObject();
    // 添加key-value
    cJSON_AddItemToObject(subObj, "factory", cJSON_CreateString("一汽大众"));
    cJSON_AddItemToObject(subObj, "last", cJSON_CreateNumber(31));
    cJSON_AddItemToObject(subObj, "price", cJSON_CreateNumber(83));
    cJSON_AddItemToObject(subObj, "sell", cJSON_CreateNumber(49));
    cJSON_AddItemToObject(subObj, "sum", cJSON_CreateNumber(80));

    // 创建json数组
    cJSON* array = cJSON_CreateArray();
    // array添加元素
    cJSON_AddItemToArray(array, cJSON_CreateNumber(123));
    cJSON_AddItemToArray(array, cJSON_CreateBool(1));
    cJSON_AddItemToArray(array, cJSON_CreateString("hello, world"));

    // 数组中的对象
    cJSON* subsub = cJSON_CreateObject();
    cJSON_AddItemToObject(subsub, "梅赛德斯奔驰", 
                          cJSON_CreateString("心所向, 持以恒"));
    cJSON_AddItemToArray(array, subsub);

    cJSON_AddItemToObject(subObj, "other", array);

    // obj中添加key - value
    cJSON_AddItemToObject(obj, "奔驰", subObj);

    // 数据格式化
    char* data = cJSON_Print(obj);
    FILE* fp = fopen("car.json", "w");
    fwrite(data, sizeof(char), strlen(data)+1, fp);
    fclose(fp);

    return 0;
}
```





#### 解析json文件

```c
○ 将字符串解析为JSON结构
cJSON *cJSON_Parse(const char *value);
返回值需要使用cJSON_Delete释放


○ 根据键值查找json节点
cJSON *cJSON_GetObjectItem(
    cJSON *object, // 当前json对象
	const char *string // key值
);

○ 获取json数组中元素的个数
int cJSON_GetArraySize(cJSON *array);

○ 根据数组下标找到对应的数组元素
cJSON *cJSON_GetArrayItem(cJSON *array, int index);

○ 判断是否有可以值对应的键值对
int cJSON_HasObjectItem(cJSON *object, const char *string);
```



解析代码：

```c
#include <stdio.h>
#include <string.h>
#include "cJSON.h"

int main(int argc, const char* argv[])
{
    if(argc < 2)
    {
        printf("./a.out jsonfile\n");
        return 0;
    }

    // 加载json文件 
    FILE* fp = fopen(argv[1], "r");
    char buf[1024] = {0};
    fread(buf, 1, sizeof(buf), fp);
    cJSON* root = cJSON_Parse(buf);

    cJSON* subobj = cJSON_GetObjectItem(root, "奔驰");
    // 判断对象是否存在
    if( subobj )
    {
        // 获取子对象
        cJSON* factory = cJSON_GetObjectItem(subobj, "factory");
        cJSON* last = cJSON_GetObjectItem(subobj, "last");
        cJSON* price = cJSON_GetObjectItem(subobj, "price");
        cJSON* sell = cJSON_GetObjectItem(subobj, "sell");
        cJSON* sum = cJSON_GetObjectItem(subobj, "sum");
        cJSON* other = cJSON_GetObjectItem(subobj, "other");

        // 打印value值
        printf("奔驰：\n");
        printf("    factory: %s\n", cJSON_Print(factory));
        printf("    last: %s\n", cJSON_Print(last));
        printf("    price: %s\n", cJSON_Print(price));
        printf("    sell: %s\n", cJSON_Print(sell));
        printf("    sum: %s\n", cJSON_Print(sum));

        // 打印数组内容
        printf("    other:\n");
        if(other->type == cJSON_Array)
        {
            for(int i=0; i<cJSON_GetArraySize(other); ++i)
            {
                cJSON* node = cJSON_GetArrayItem(other, i);
                // 判断数据类型
                if(node->type == cJSON_String)
                {
                    printf("        %s  \n", node->valuestring);
                }
                if(node->type == cJSON_Number)
                {
                    printf("        %d\n", node->valueint);
                }
                if(node->type == cJSON_True)
                {
                    printf("        %d\n", node->valueint);
                }
                if(node->type == cJSON_False)
                {
                    printf("        %d\n", node->valueint);
                }
            }
        }
    }

    cJSON_Delete(root);
    fclose(fp);


    return 0;
}
```

