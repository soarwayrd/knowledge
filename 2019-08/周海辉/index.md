### 一、在不改变angular默认脏检测机制下，手动优化脏检测

import { NgZone } from '@angular/core';

```javascript
//调用runOutsideAngular方法，Angular不会对里面的变化进行跟踪。
 this.zone.runOutsideAngular(()=>{});

//重新跟踪变化
 this.zone.run(()=>{});

 ```
##正常模式下
https://stackblitz.com/edit/angular-j5ox5x?file=src/app/app.component.ts

```javascript
  for(let i = 1;i<100;i++)
   {
     this.timer2 = setInterval(()=>{
      if(this.sum > 10000)
        {
          window.clearInterval(this.timer2);
        }
      else
        this.sum ++;
    },100);
   }
```

##使用zone模式下
https://stackblitz.com/edit/angular-frep1w?file=src/app/app.component.ts

```javascript
  this.zone.runOutsideAngular(()=>{
    for(let i = 1;i<100;i++)
   {
      this.timer = setInterval(()=>{
      if(this.count > 10000)
        {
          window.clearInterval(this.timer);
          this.zone.run(()=>{
            this.count = this.count;
          });
        }
      else
        this.count ++;
    },10);
   }
  });
```