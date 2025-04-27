# component-store

---

## 概念

>状态管理器，想象成localstorage，然后可以做各种处理，类似于创建了一个用来存储数据的服务，然后有类似subject的属性，可以订阅和发布，依托于ngRx，比ngRx轻量化，在组件中使用

## 属性和方法

### state

>状态值，类似一个大的对象

### updater

>更新状态的数据

### select/selectSignal

>从[state](#state)中切分数据，如 ***{ propertyA , propertyB }***，使用*select*或者*selectSignal*可以切分出propertyA，*selectSignal*可以将数据转换成[Signal](../angular/signals.markdown)对象

### effect

>处理异步请求的地方，会结合rxjs中的**tap**操作符处理异步请求并在异步请求中更新[state](#state)，然后返回一个**Observable**对象，示例如下：

```ts
readonly loadUsers = this.effect<void>(
    // 使用管道操作符组合逻辑
    (trigger$) => trigger$.pipe(
      tap(() => this.patchState({ loading: true })), // 更新加载状态
      switchMap(() => 
        this.userService.getUsers().pipe(
          tap({
            next: (users) => this.patchState({ users, loading: false }), // 成功
            error: (error) => this.patchState({ loading: false }),       // 失败
          }),
          catchError((error) => {
            console.error('加载失败:', error);
            return of(null); // 防止流中断
          })
        )
      )
    )
)
```

## 监听属性的变化

>将[select](#selectselectsignal)处理过后的值转换成**Observable***对象后可以监听对应的变化

## 使用细节

>创建好store之后要注入到需要使用的组件中  
>生命周期和注入的组件相关，不同的独立组件不共享状态，如果需要共享，需要在父组件注入
