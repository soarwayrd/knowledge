# 前端嵌套对象访问模式 #

## Angular嵌套对象访问模式 ##

Angular 的安全导航操作符 (?.) ，用来保护出现在属性路径中 null 和 undefined 值。 

```html
{{currentHero?.name}}
```

## JS嵌套对象访问模式 ##

```javascript
let a = null;
a.Name
Uncaught TypeError: Cannot read property 'Name' of null
(a || {}).Name
undefined

let b;
b.Name
Uncaught TypeError: Cannot read property 'Name' of undefined
(b || {}).Name
undefined
```
使用这种表示法，永远不会遇到无法读取未定义的属性的属性。做法是检查用户是否存在，如果不存在，就创建一个空对象，这样，下一个级别的键将始终从存在的对象访问。