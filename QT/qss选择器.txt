### 1，通用选择器（*）

### 2，类型选择器（类名）

类名 作为选择器，作用于它自己和它的所有子类。

### 3，类选择器（.类名）

会作用于它自己，它的子类不受影响，注意和类型选择器的区别。（和类型选择器是有区别的）

### 4，ID 选择器（#对象名）

+objectName 作为选择器，只作用于用此 objectName 的对象

### 5，属性选择器（[属性="值"]）

选择器[属性="值"] 作为选择器，这个属性可用通过 object->property(propertyName) 访问的，Qt 里称为 Dynamic Properties。

### 6，包含选择器（类名 类名 类名。。。 类名之间用空格隔开）

选择器之间用空格隔开

### 7，子元素选择器（>）

选择器之间用 > 隔开，作用于 Widget 的直接子Widget，注意和包含选择器的区别。

### 8，伪类选择器（:）

选择器:状态 作为选择器，支持 ! 操作符，表示 非。QPushButton:hover { color: white }

### 9，Subcontrol 选择器

选择器::subcontrol 作为选择 Subcontrol 的选择器。

```
QCheckBox::indicator
{
    width: 20px;
    height: 20px;
}
```

