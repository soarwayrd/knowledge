### JS 解构赋值

let arr = [1,2,3];
let a,b,c;
[a,b,c] = arr;
console.log(a,b,c);//1 2 3

let name='alpha',age=12;
let obj = {};
obj = {name,age};
console.log(obj);//Object {
  age: 12,
  name: "alpha"
}

let {name:alias} = obj; 
console.log(alias);//"alpha"