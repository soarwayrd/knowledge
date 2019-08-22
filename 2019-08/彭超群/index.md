### 一、快速删除node_modules
```
npm install rimraf -g 

rimraf node_modules
```
github：https://github.com/isaacs/rimraf


### 二、登录页背景动画
[在线演示](https://vincentgarreau.com/particles.js/)

用法
```typescript
// Import ParticlesModule
import { ParticlesModule } from 'angular-particle';

@NgModule({
  declarations: [
    ...
  ],
  imports: [
	...
    ParticlesModule
  ],
  providers: [],
  bootstrap: []
})
export class AppModule { }
```
```typescript
<particles class="particles" [params]="myParams" [style]="myStyle" [width]="width" [height]="height"></particles>
```

``` typescript
this.myStyle = {
      'position': 'fixed',
      'width': '100%',
      'height': '100%',
      'z-index': -1,
      'top': 0,
      'left': 0,
      'right': 0,
      'bottom': 0,
      'background-color': '#F4F6F9' // 夜用 #232741  日用 #F4F6F9 
    };
    this.myParams = {
      "particles": {
        "number": {
          "value": 160,
          "density": {
            "enable": true,
            "value_area": 1000
          }
        },
        "color": {
          "value": "#40a9ff"
        },
        "shape": {
          "type": "circle",
          "stroke": {
            "width": 0,
            "color": "#40a9ff"
          },
          "polygon": {
            "nb_sides": 5
          },
          "image": {
            "src": "img/github.svg",
            "width": 100,
            "height": 100
          }
        },
        "opacity": {
          "value": 0.25,
          "random": true,
          "anim": {
            "enable": true,
            "speed": 1,
            "opacity_min": 0.1,
            "sync": false
          }
        },
        "size": {
          "value": 6,
          "random": true,
          "anim": {
            "enable": false,
            "speed": 3,
            "size_min": 0.5,
            "sync": false
          }
        },
        "line_linked": {
          "enable": true,
          "distance": 100,
          "color": "#ddd",
          "opacity": 0.9,
          "width": 1
        },
        "move": {
          "enable": true,
          "speed": 1.5,
          "direction": "none",
          "random": true,
          "straight": false,
          "out_mode": "out",
          "bounce": false,
          "attract": {
            "enable": false,
            "rotateX": 600,
            "rotateY": 600
          }
        }
      },
      "interactivity": {
        "detect_on": "canvas",
        "events": {
          "onhover": {
            "enable": true,
            "mode": "grab"
          },
          "onclick": {
            "enable": true,
            "mode": "bubble"
          },
          "resize": true
        },
        "modes": {
          "grab": {
            "distance": 400,
            "line_linked": {
              "opacity": 1
            }
          },
          "bubble": {
            "distance": 250,
            "size": 0,
            "duration": 2,
            "opacity": 0,
            "speed": 3
          },
          "repulse": {
            "distance": 400,
            "duration": 0.4
          },
          "push": {
            "particles_nb": 4
          },
          "remove": {
            "particles_nb": 2
          }
        }
      },
      "retina_detect": true
    }
```




### 三、foreach用throw 中断
return无法跳出forEach
```typescript
 array.forEach(e => {
    if (condition) {
      return;
    }
  });
```

但是可以抛出一个throw来控制
```typescript
try { 
  array.forEach(e => {
    if (condition) {
      throw "有空值"
    }
  });
} catch (error) {
  return;
}
```