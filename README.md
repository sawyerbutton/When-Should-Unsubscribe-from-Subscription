# When-Should-Unsubscribe-from-Subscription

## 在Angular中何时应当注销订阅

> 曾经在官方路由文档中有这么一段话和示例

> _Eventually, we'll navigate somewhere else. The router will remove this component from the DOM and destroy it. We need to clean up after ourselves before that happens. Specifically, we must unsubscribe before Angular destroys the component. Failure to do so could create a memory leak.We unsubscribe from our Observable in the ngOnDestroy method._

```typescript
private sub: any;

ngOnInit() {
  this.sub = this.route.params.subscribe(params => {
     let id = +params['id']; // (+) converts string 'id' to a number
     this.service.getHero(id).then(hero => this.hero = hero);
   });
}

ngOnDestroy() {
  this.sub.unsubscribe();
}
```

> 是当路由跳转到新的页面时,路由将会从DOM中移除当前组件,在销毁组件之前必须清理组件本身,具体来说必须在Angular销毁组件之前取消组件中的订阅-`如果不这样做可能会造成内存泄漏`(需要在`ngOnDestroy`方法中取消订阅的`Observable`)

> 在Angular V4的版本中更新了关于路由订阅的说明:`路由器管理其所提供的可观察对象并实现订阅本地化当组件被销毁时将会清理订阅以防止内存泄漏,因此不需要手动取消订阅`_route params Observable_

> 这些说明并没有明确解释在何时何处对以及为何需要对已订阅的Observable对象取消订阅

### `官方承认`的`解决方案`(2017-4)

> 先说方案

- 在组件中添加一个私有的`Observable(一个subject)`,这里的组件范围包括了所有调用了`.subcribe()`方法订阅`Observable`的的类

```typescript
private ngUnsubscribe = new Subject();
```

- 在`ngOnDestroy`钩子中调用`this.ngUnsubscribe.next()`和`this.ngUnsubscribe.complete()`方法

- 真正的秘籍是在每个`.subscribe()`方法前链式调用`takeUntil(this.ngUnsubscribe)`方法,配合`ngOnDestroy`钩子中的方法将保证在销毁组件时清理所有订阅

```typescript
import { Component, OnDestroy, OnInit } from '@angular/core';
// RxJs 6.x+ import paths
import { filter, startWith, takeUntil } from 'rxjs/operators';
import { Subject } from 'rxjs';
import { BookService } from '../books.service';

@Component({
    selector: 'app-books',
    templateUrl: './books.component.html'
})
export class BooksComponent implements OnDestroy, OnInit {
    private ngUnsubscribe = new Subject();

    constructor(private booksService: BookService) { }

    ngOnInit() {
        this.booksService.getBooks()
            .pipe(
               startWith([]),
               filter(books => books.length > 0),
               takeUntil(this.ngUnsubscribe)
            )
            .subscribe(books => console.log(books));

        this.booksService.getArchivedBooks()
            .pipe(takeUntil(this.ngUnsubscribe))
            .subscribe(archivedBooks => console.log(archivedBooks));
    }

    ngOnDestroy() {
        this.ngUnsubscribe.next();
        this.ngUnsubscribe.complete();
    }
}
```

> 值得注意的是,将`takeUntil`运算符添加为`最后一个运算符`非常重要,以防止运算符链中的`中间某层Observable对象`泄漏

> 如果`takeUntil`运算符放在涉及订阅的另一个Observable源的运算符之前,当`takeUntil`收到其`complete`通知时,可能不会取消总源的订阅,比如

```typescript
import { Observable } from "rxjs";
import { combineLatest, takeUntil } from "rxjs/operators";

declare const a: Observable<number>;
declare const b: Observable<number>;
declare const notifier: Observable<any>;

const c = a.pipe(
  takeUntil(notifier),
  combineLatest(b)
).subscribe(value => console.log(value));
```

```typescript
import { Observable } from "rxjs";
import { switchMap, takeUntil } from "rxjs/operators";

declare const a: Observable<number>;
declare const b: Observable<number>;
declare const notifier: Observable<any>;

const c = a.pipe(
  takeUntil(notifier),
  switchMap(_ => b)
).subscribe(value => console.log(value));
```

> 理论上当`notifier`程序发出时,`takeUntil`运算符返回的Observable完成,自动取消订阅任何订阅者

> 但是`c的订阅者`没有订阅takeUntil返回的`Observable`而是订阅了`combineLatest`或`switchMap`返回的Observable ,所以它不会在`takeUntil` observable完成后自动取消订阅

> `c的订阅者`将保持订阅直到所有Observable传递给`combinedLast`或`switchMap`完成,因此,除非在`notifier`发出之前完成b,否则对b的订阅将泄漏

> 通过将`takeUnitl`放置在操作符的末端,当`notifier`发出时,c的订阅者将被自动取消订阅 ,因为takeUntil返回的Observable将完成并且takeUntil的实现将取消订阅`combineLatest`返回的Observable,以实现取消订阅a和b

> 如果正在使用`takeUntil`机制来实现间接取消订阅,则可以通过启用官方已添加到`rxjs-tslint-rules`包中的`rxjs-no-unsafe-takeuntil`规则来确保`takeUntil`始终是传递给管道的最后一个运算符

### 最新的一些方案和说法(自行斟酌)

### 使用功能组件和展示组件构建组件架构

> 如果在组件中过分使用`takeUntil()`方法可能意味着当前组件承担了过多的责任和义务,应当考虑将当前组件拆分为`feature component`功能组件和`presentational conponent`展示组件

> 利用`async` 管道将Observable从功能组件转移到展示组件的`Input`属性中以避免在任何地方使用`unsubscribe()`方法

```typescript
import { Component, OnDestroy, OnInit } from '@angular/core';
// RxJs 6.x+ import paths
import { filter, startWith, takeUntil } from 'rxjs/operators';
import { Subject } from 'rxjs';
import { BookService } from '../books.service';

@Component({
    selector: 'app-books',
    templateUrl: `
        <book [book] = "book$ | async"></book>
        <archived-books [archivedBooks] = "archivedBooks$ | async"></archived-books>
    `
})
export class BooksComponent implements OnDestroy, OnInit {
    // private ngUnsubscribe = new Subject();
    private book$: any;
    private archivedBooks$: any
    constructor(private booksService: BookService) { }

    ngOnInit() {
        this.book$ = this.booksService.getBooks()
            .pipe(
               startWith([]),
               filter(books => books.length > 0)
            );

        this.archivedBooks$ = this.booksService.getArchivedBooks();
    }
}
```

#### 何为`smart component`&`presentional component`

> 可能存在的困惑有什么是`feature/presenataional component`

> 在考虑Angular应用程序的架构时最容易考虑到的是尽可能将每个事物切分为组件,但是随之而来的带来了另外一些困惑

- 存在哪些类型的组件
- 组件间如何交互
- 是否允许将服务插入任何一个组件中
- 如何让组件可复用

> 为了解决上述问题需要将组件拆分为两个概念

1. 智能/功能组件(`feature component`)：有时也称为应用程序级组件,容器组件或控制器组件
2. 展示组件(`presentational component`)：有时也称为纯组件或无知组件

> 使用一个未拆分的例子来描述常规的组件使用

```typescript
@Component({
  selector: 'app-home',
  template: `
    <h2>All Lessons</h2>
    <h4>Total Lessons: {{lessons?.length}}</h4>
    <div class="lessons-list-container v-h-center-block-parent">
        <table class="table lessons-list card card-strong">
            <tbody>
            <tr *ngFor="let lesson of lessons" (click)="selectLesson(lesson)">
                <td class="lesson-title"> {{lesson.description}} </td>
                <td class="duration">
                    <i class="md-icon duration-icon">access_time</i>
                    <span>{{lesson.duration}}</span>
                </td>
            </tr>
            </tbody>
        </table>
    </div>
`,
  styleUrls: ['./home.component.css']
})
export class HomeComponent implements OnInit {

    lessons: Lesson[];

  constructor(private lessonsService: LessonsService) {
  }

  ngOnInit() {
      this.lessonsService.findAllLessons()
          .do(console.log)
          .subscribe(
              lessons => this.allLessons = lessons
          );
  }

  selectLesson(lesson) {
    ...
  }

}
```

> 即使这个组件的逻辑并不复杂,但是也可以看出他有逐渐变得臃肿的趋势

> 值得考虑的是,上述代码中创造的课程列表功能可能在别的页面中也会有类似的需求需要实现

> 这种情况下使这个功能变得可复用是唯一可选的方案,当前将课程列表这个功能抽离出来单独做成组件

```typescript
import {Component, OnInit, Input, EventEmitter, Output} from '@angular/core';
import {Lesson} from "../shared/model/lesson";

@Component({
  selector: 'lessons-list',
  template: `
      <table class="table lessons-list card card-strong">
          <tbody>
          <tr *ngFor="let lesson of lessons" (click)="selectLesson(lesson)">
              <td class="lesson-title"> {{lesson.description}} </td>
              <td class="duration">
                  <i class="md-icon duration-icon">access_time</i>
                  <span>{{lesson.duration}}</span>
              </td>
          </tr>
          </tbody>
      </table>  
  `,
  styleUrls: ['./lessons-list.component.css']
})
export class LessonsListComponent {

  @Input()
  lessons: Lesson[];

  @Output('lesson')
  lessonEmitter = new EventEmitter<Lesson>();

    selectLesson(lesson:Lesson) {
        this.lessonEmitter.emit(lesson);
    }

}
```

> 在`LessonsListComponent`组件中并没有插入任何的service用以获取数据,而是通过`@Input()`修饰符从父组件获取数据

> 这样的做法的好处是组件本身并不在乎数据从何而来,组件的责任纯粹是向用户呈现数据而不包括从特定位置获取数据,这就是一个很典型的展示组件

> 此时相应的主组件`home component`也需要做出相应的修改

```typescript
import { Component, OnInit } from '@angular/core';
import {LessonsService} from "../shared/model/lessons.service";
import {Lesson} from "../shared/model/lesson";

@Component({
  selector: 'app-home',
  template: `
      <h2>All Lessons</h2>
      <h4>Total Lessons: {{lessons?.length}}</h4>
      <div class="lessons-list-container v-h-center-block-parent">
          <lessons-list [lessons]="lessons" (lesson)="selectLesson($event)"></lessons-list>
      </div>
`,
  styleUrls: ['./home.component.css']
})
export class HomeComponent implements OnInit {

    lessons: Lesson[];

  constructor(private lessonsService: LessonsService) {
  }

  ngOnInit() {
     ...
  }

  selectLesson(lesson) {
    ...
  }

}
```

> 主组件中的列表展示部分被替换成了`LessonsListComponent`组件,整体的展示效果并没有发生变化,主组件在仍然知晓如何从service中获取数据的同时,移除了向用户展示数据的需求

> 在这种情况下,主组件就可以称之为一个功能组件/智能组件,这一类组件和应用本身相绑定难以用于其他的应用中,类似于DDD中的应用层

> 视图的顶级组件可能永远是智能组件,即使在组件中将数据加载转移到路由器数据解析器,该组件仍然至少必须将`ActivatedRoute`服务注入其中,这样的特性决定了一个应用中至少存在一个功能/智能组件

##### 功能组件和展示组件之间的交互

> 简而言之,让功能组件通过`@Input`属性将数据注入展示组件,并接收展示组件通过`@Output`触发的任何操作

> 对于使用`@Output`抛出事件而言,展示组件通过清晰定义的接口实现与功能组件的隔离

- 展示组件抛出一个事件,但是并不知晓是谁接收了这一事件以及会对这一事件进行何种操作
- 功能组件订阅由展示组件抛出的事件,但是他并不知道这一事件是通过何种方式触发的(可能是用户在展示组件中点击了某个按钮或其他操作)
- 上述这些对于功能组件是完全透明的,也造就了功能组件的干净

> 只是随之而来的是另一个问题,如果某展示组件与功能组件之间存在其他展示组件,因为`@Output`只能向唯一上级传递事件,而展示组件并没有接收事件的能力,从底层展示组件抛出的事件将会无法传递到功能组件之中

> 遇到这样的情况会很让人困惑,为何不创造一个自定义冒泡事件处理类似的状况呢,

> 其实这并非偶然而是归于设计理念上`避免使用类似于AngularJS中使用$scope.$emit()和$scope.$broadcast()处理事件混乱的问题`

> `AngularJS`中的`event bus`机制最终会导致应用中本不应该知晓对方的部分产生强依赖,事件以难以理解的方式互相影响

> `展示组件的自定义事件只能由其父组件显示,而不能在后续的组件树种使用`的设计理念源自于此

> 如果由于某种原因确实需要自定义冒泡行为仍然可以使用`element.dispatchEvent()`实现

_`除非非不得已不要使用这样的方式`_

> 正确的解决方案

1. 在大型项目中使用`ngrx/store`进行状态管理(但即使存在`stroe`,某些情况下也可能不希望在展示组件中注入`store`)

- 为了简化示例创建一个`store`服务来解决上述的课程选择问题

```typescript
@Injectable()
export class LessonSelectedService {

    private _selected: BehaviorSubject<Lesson> = new BehaviorSubject(null);

    public selected$ = this._selected.asObservable().filter(lesson => !!lesson);
    select(lesson:Lesson) {
         this._selected.next(lesson);
    }
}
```

`LessonSelectedService`服务暴露了一个Observable对象`selected$`用于每次新的课程被选择后抛出该课程

- _值得注意的是,在服务内创建了没有暴露给外部的BehaviorSubject对象,这是因为设计时希望`BehaviorSubject`对象作为一个事件总线而存在,用以控制服务内抛出事件的角色_

- _如果`LessonSelectedService`服务向外部暴露了`BehaviorSubject`就会导致应用的任何一个部分都可以代表该服务抛出事件,这违背了封装的法则_

- 如何在主组件中插入该服务

```typescript
@Component({...})
export class HomeComponent implements OnInit {

    lessons: Lesson[];

  constructor(
        private lessonsService: LessonsService, 
        private lessonSelectedService: LessonSelectedService) {
  }

  ngOnInit() {
      // ....
      // 通过订阅服务的BehaviorSubject订阅最新的lesson数据
      this.lessonSelectedService.selected$.subscribe(lesson => this.selectLesson(lesson));
  }

  selectLesson(lesson) {
    ...
  }

}
```

> 需要注意的是主组件不知道`lesson list`只知道应用程序的其他部分触发了`lesson list`,应用程序的两个部分仍然是隔离的

- 选择器的抛出和`home component`毫无关联,`home component`和并不知道课程数据,两者之间通过`LessonSelectedService`交流

- 即使已经知晓了如何在`home component`中注入使用`LessonSelectedService`, 如何在不破坏展示组件`展示属性`的前提下使用`LessonSelectedService`仍然是一个问题,为了解决这个问题有两种方案

1. 第一种不那么体面,让问题不再成为问题即可(`将展示组件转换为带有功能逻辑的组件`),比如某些情况下可能具体的组件会作为一个全应用级别的使用对象的情况 -> 其实这种情况是很常见的,对于一个应用而言,功能组件不仅仅存在于结构的顶端也存在于树形结构的下层中依旧是可以接受的方案

2. 解决问题的另一种方法是保持展示组件不变,并在需要的地方使用它,只是在使用时将它包装在一个功能组件中,并在该包装性质的功能组件中注入所需服务

```typescript
@Component({
  selector: 'custom-lessons-list',
  template: `
       <lessons-list [lessons]="lessons" (lesson)="selectLesson($event)"></lessons-list>  
  `
})
export class CustomLessonsListComponent {

   constructor(private lessonSelectedService: LessonSelectedService) {

   }

   selectLesson(lesson) {
       this.lessonSelectedService.select(lesson); 
   }

}
```

> 在使用功能组件和展示组件分割项目时时刻询问自己

1. 这种表示逻辑在应用程序的其他地方是否有用
2. 将解构进一步分解会有用吗
3. 否在应用中创建了意外的紧耦合

> 合理地推进重构过程是分割项目的最佳实践

#### 希望你还记得本文的主题是如何处理订阅的问题,虽然上面说了可以拉出来当做单独主题的内容,但是就像刚刚说的,文字的撰写也是不断重构的过程,现在就让我们回到解决方案这个环节上

#### ngx-take-until-destroy

> 上面提到过官方支持`takeUntil`操作符将其放置到所有订阅管道的最后来保证Observable订阅泄露的问题

> 同样的方式可以使用npm包[`ngx-take-until-destroy`](https://github.com/NetanelBasal/ngx-take-until-destroy)

```bash
npm install ngx-take-until-destroy --save
```

```typescript
// 对于Angular
import { untilDestroyed } from 'ngx-take-until-destroy';

@Component({
  selector: 'app-inbox',
  templateUrl: './inbox.component.html'
})
export class InboxComponent implements OnInit, OnDestroy {
  ngOnInit() {
    interval(1000)
      .pipe(untilDestroyed(this))
      .subscribe(val => console.log(val));
  }

  // This method must be present, even if empty.
  ngOnDestroy() {
    // To protect you, we'll throw an error if it doesn't exist.
  }
}
```

```typescript
// 对于任何class
import { untilDestroyed } from 'ngx-take-until-destroy';

export class Widget {
  constructor() {
    interval(1000)
      .pipe(untilDestroyed(this, 'destroy'))
      .subscribe(console.log);
  }

  // The name needs to be the same as the decorator parameter
  destroy() {}
}
```

#### DIY订阅处理指令

> 在Angular模板中订阅Observable是通过`async`管道实现的

```html
<section>
  
  <app-alerts [alerts]="alerts$ | async"></app-alerts>
  
  <div>
    <other-component [alerts]="alerts$ | async"></other-component>
  </div>
  
</section>
```

> 上述代码的问题是我们在`alert$`Observable对象上创建了多个订阅，每个异步管道都有一个

> Angular提供了一个通过`*ngIf`的解决方案处理上述状况

```html
<section *ngIf="alerts$ | async as alerts">
  
  <app-alerts [alerts]="alerts"></app-alerts>
  
  <div>
    <other-component [alerts]="alerts"></other-component>
  </div>
  
</section>
```

- 这样操作的好处在于
1. 只创建了一个`async`管道并只创建了一次订阅的过程
2. 本地变量`alert`以更合理的方式重复使用

> 使用这样的方案当然是无可厚非也没有错误，但是存在一些`舒适度的问题`

1. 代码语义意义丢失了，代码没有正确地表达它的含义：分不清是进行条件渲染，或者只需将表达式的结果传递给子视图
2. 如果`*ngIf`的值为falsy，则在`ngIf`指令中使用它意味着将不会完全呈现视图,这可能不是期望的结果

> 出于这些原因创建一个解决上述问题的指令(目的是订阅一个observable并将结果公开给它的子视图)是很有意义的

```html
<section *ngSubscribe="alerts$ as alerts">
  
  <app-alerts [alerts]="alerts"></app-alerts>
  
  <div>
    <other-component [alerts]="alerts"></other-component>
  </div>
  
</section>
```

> 类似于`*ngIf`，`*ngSubscribe`也是一个结构型指令

```typescript
export class NgSubscribeContext {
  $implicit = null;
  ngSubscribe = null;
}

@Directive({
  selector: '[ngSubscribe]'
})
export class NgSubscribeDirective implements OnInit, OnDestroy {

  private observable: Observable<any>;
  private context: SubscribeContext = new NgSubscribeContext();
  private subscription: Subscription;

@Input()
  set ngSubscribe(inputObservable: Observable<any>) {
    if (this.observable !== inputObservable) {
      this.observable = inputObservable;
      // 重置订阅
      this.subscription && this.subscription.unsubscribe();
      this.subscription = this.observable.subscribe(value => {
        this.context.ngSubscribe = value;
        // markForCheck用于应用onPush策略
        this.cdr.markForCheck();
      });
    }

  constructor(
    private viewContainer: ViewContainerRef,
    private cdr: ChangeDetectorRef,
    private templateRef: TemplateRef<any>
  ) {}

  ngOnInit() {
    this.viewContainer.createEmbeddedView(this.templateRef, this.context);
  }

  ngOnDestroy() {
    // 取消订阅
    this.subscription && this.subscription.unsubscribe();
  }
}
```

> 需要订阅输入Observable，通过context将结果传递给模板，值得注意的是调用`markForCheck`来支持采取了`OnPush`策略的组件

> 在模板中使用的`as alerts`实际上绑定到了`ngSubscribe`的`contest`中

> 利用这样的方式`*ngSubscribe`成为了在模板中订阅数据更好地指令方案