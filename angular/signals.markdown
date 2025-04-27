# signal

---

基于angular17的版本（写于2025.04.27）

## 概念

>像一个对象，它本身不会变，变的是它的值，用到的地方类似js按址引用，所以更新值之后，全都会变，对于组件来说，只有数据被重新渲染了，可以结合**angular**组件变更的**Onpush**一起使用

## 属性和方法

### 创建Signal

#### Signal()

>使用 **signal()** 创建 **Signal** 对象，**Signal** 对象包括 *writeable* 和 *readonly* 两种，简单的代码如下：

```ts
const count = signal(0);

//还可以自定义相等条件
import _ from 'lodash';

const data = signal(['test'], {equal: _.isEqual});

// Even though this is a different array instance, the deep equality
// function will consider the values to be equal, and the signal won't
// trigger any updates.
data.set(['test']);
```

#### computed()

>使用 **computed()** 创建：

```ts
const doubleCount = computed(()=>{
    count() * 2;
})
```

>当 ***count*** 的值变化时， ***doubleCount*** 的值会监听到 ***count*** 变化，然后跟着一起变化值  
>
>**computed()** 是懒加载和存储的，当 **computed()** 中的依赖项没有更新时，调用 **computed()** 会得到之前计算过后存储的值，当依赖更新时，存储的值才会惊醒修改

#### input()

>父组件传值给子组件时可以用angular的新特性 **input()** 在子组件创建 **Signal** 对象

```ts
import {Component, input} from '@angular/core';
@Component({...})
export class MyComp {
    // optional
    firstName = input<string>();         // InputSignal<string|undefined>
    age = input(0);                      // InputSignal<number>
    // required
    lastName = input.required<string>(); // InputSignal<string>
}
```

>使用 **input()** 创建的 **Signal** 对象都是 *readonly* 的，不可以更新 **Signal** 对象的值  
>**inputSignal** 还有两个属性，分别是 **transform** 和 **alias**

##### transform

>使用 **transform** 来格式化输入数据

```ts
class MyComp {
    disabled = input(false, {
        transform: (value: boolean|string) => typeof value === 'string' ? value === '' : value,
    });
}
```

##### alias

>使用 **alias** 在子组件创建别名，父组件输入的是 *studentAge* ，子组件使用 *age* 调用 *studentAge*

```ts
class StudentDirective {
    age = input(0, {alias: 'studentAge'});
}
//父组件使用[studentAge]传值

```

#### model()

>angular新特性，同样是用在父组件传值到子组件上

```ts
import {Component, model, input} from '@angular/core';

@Component({
    selector: 'custom-checkbox',
    template: '<div (click)="toggle()"> ... </div>',
})
export class CustomCheckbox {
    checked = model(false);
    disabled = input(false);

    toggle() {
    // While standard inputs are read-only, you can write directly to model inputs.
    this.checked.set(!this.checked());
}
}
```

>**model()** 产生的 **Signal** 对象是可以更新值的，可以依据此特性实现双向绑定

```ts
@Component({
    ...,
    // `checked` is a model input.
    // The parenthesis-inside-square-brackets syntax (aka "banana-in-a-box") creates a two-way binding
    template: '<custom-checkbox [(checked)]="isAdmin" />',
})
export class UserProfile {
    protected isAdmin = false;
}

@Directive({...})
export class CustomCheckbox {
    // This automatically creates an output named "checkedChange".
    // Can be subscribed to using `(checkedChange)="handler()"` in the template.
    checked = model(false);
}
```

### 监听Signal的值

#### computed()监听

>[**computed()**](#computed) 能监听变化的原因是对 **Signal** 对象产生了依赖，所以要监听到 **Signal** 的变化，需要建立依赖

```ts
//这种写法能先与showCount()创建依赖，因为判断的时候执行了showCount()，
// 但是无法与count（）创建依赖，所以count()变化时，会无法执行这段代码，
// 当执行了count()之后，count()的值有变化时也会执行以下逻辑
const showCount = signal(false);
const count = signal(0);
const conditionalCount = computed(() => {
    if (showCount()) {
        return `The count is ${count()}.`;
    } else {
        return 'Nothing to see here!';
    }
});
```

>引用angular官方文档中的解释  
>Note that dependencies can be removed as well as added. If showCount is later set to false again, then count will no longer be considered a dependency of conditionalCount

#### effect()

>有点类似 **ngOnChange()** ，**effect()** 会监听所有 **Signal** 的变化

```ts
effect(() => {
    console.log(`The current count is: ${count()}`);
});
```

>以下是官方推荐的使用场景  
>
>1. Logging data being displayed and when it changes, either for analytics or as a debugging tool  
>2. Keeping data in sync with window.localStorage  
>3. Adding custom DOM behavior that can't be expressed with template syntax  
>4. Performing custom rendering to a \<canvas\>, charting library, or other third party UI library

#### toObservable()

>和rxjs一起使用，将 **Signal** 对象转换成 **Observable** 对象，但和一般的 **Observable** 对象不同，他的底层实现原理是通过 [**effect**](#effect) 实现的，所以 [**effect**](#effect) 的特性他都有

```ts
import { Component, signal } from '@angular/core';
import { toObservable } from '@angular/core/rxjs-interop';

@Component(...)
export class SearchResults {
    query: Signal<string> = inject(QueryService).query;
    query$ = toObservable(this.query);

    results$ = this.query$.pipe(
        switchMap(query => this.http.get('/search?q=' + query ))
    );
}
```

>官方解释中，有一个点需要注意，以下为官方文档  
>
>Unlike Observables, signals never provide a synchronous notification of changes. Even if your code updates a signal's value multiple times, effects which depend on its value run only after the signal has "settled"

```ts
const obs$ = toObservable(mySignal);
obs$.subscribe((value) => console.log(value));

mySignal.set(1);
mySignal.set(2);
mySignal.set(3);
//Here, only the last value (3) will be logged.update也是同样的道理，只会获取最后的值
```

### 取消监听

#### untracked()

>使用 **untraced()** 取消监听

```ts
effect(() => {
    console.log(`User set to `${currentUser()}` and the counter is ${untracked(counter)}`);
});

effect(() => {
    const user = currentUser();
    untracked(() => {
        // If the `loggingService` reads signals, they won't be counted as
        // dependencies of this effect.
        this.loggingService.log(`User set to ${user}`);
    });
});
```

### 更新Signal的值

#### update()/set()

```ts
count.set(3);

// Increment the count by 1.
count.update(value => value + 1);
```
