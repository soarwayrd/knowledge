### JS 解构赋值
```javascript
let arr = [1,2,3];
let a,b,c;
[a,b,c] = arr;
console.log(a,b,c);//1 2 3


let person = {
    name: '达龙 福克斯',
    age: 20,
    gender: 'male',
    passport: 'G-12345678',
    school: 'Kentucky',
    address: {
        city: '重庆',
        street: '龙湖时代天街',
        zipcode: '400002'
    }
};
let { name, address: { city, zip } } = person;


let name='alpha',age=12;
let obj = {};
obj = {name,age};
console.log(obj);//Object {
  age: 12,
  name: "alpha"
}
```
# 简化JS嵌套对象访问模式 #
```javascript
let a = null;
a.Name
Uncaught TypeError: Cannot read property 'Name' of null
(a || {}).Name
undefined


let {name} = a; 
console.log(name);//obj不含name属性也不会报错，只会返回undefined
```
