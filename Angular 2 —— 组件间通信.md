1. 父组件向子组件传递信息
使用@Input
子组件的属性用 @Input 进行修饰，在父组件的模板中绑定变量

例子：

import { Component, OnInit, Input } from '@angular/core';

@Component({
    selector: 'my-input',
    template: `
    <h1>I am your father!</h1>
    <p>{{content}}</p>
    <input-child [content]="content"></input-child>
    `
})

export class InputComponent implements OnInit {
    content: string = 'May the force be with you';
    ngOnInit() { }
}

@Component({
    selector: 'input-child',
    template: `
    <h1>The skywalker.</h1>
    <p>{{content}}</p>`
})
export class InputChildComponent {
    @Input() content: string = '';
}
效果：



使用 setter 拦截输入的属性
在子组件中做一些修改，添加两个私有字段：revertContent、_content，然后为 _content 添加 setter 和 getter

代码

@Component({
    selector: 'input-child',
    template: `
    <h1>The skywalker.</h1>
    <p>{{content}}</p>
    <p>{{revertContent}}</p>`
})
export class InputChildComponent {
    revertContent: string;
    _content: string = '';
    @Input() set content(str: string) {
        this._content = str;
        this.revertContent = str.split(' ').reverse().join(' ');
    }

    get content() {
        return this._content;
    }
}
效果：



使用 ngOnChanged 拦截输入的属性
首先要修改一下 import 内容：

import { Component, OnInit, Input, OnChanges, SimpleChange } from '@angular/core';
然后让子组件实现 OnChanges 接口的 ngOnchanges 方法，修改后的子组件如下：

@Component({
    selector: 'input-child',
    template: `
    <h1>The skywalker.</h1>
    <p>{{content}}</p>
    <p>{{revertContent}}</p>`
})
export class InputChildComponent implements OnChanges {
    revertContent: string = 'default content';
    _content: string = 'default content';

    @Input() set content(str: string) {
        this._content = str;
    }

    get content() {
        return this._content;
    }

    ngOnChanges(changes: { [propKey: string]: SimpleChange }) {
        if (changes['content'] !== undefined) {
            let value = <string>(changes['content'].currentValue);
            this.revertContent = value.split(' ').reverse().join(' ');
            this.revertContent = `revert "${value}" to "${this.revertContent}"`;
        }
    }
}
ngOnChanges 是一个生命周期钩子，下图指出了这个方法的调用时机。


效果：



2. 子组件向父组件传递消息
子组件使用 EventEmitter<TEvent>
子组件中声明一个 EventEmitter<TEvent> 属性，然后在父组件的模板中添加这个事件的监听器

代码：

import { Component, OnInit, Output, EventEmitter } from '@angular/core';

@Component({
    selector: 'event',
    template: `
    <h1>I'm your father.</h1>
    <p>recived damage: {{damage}}</p>
    <event-child (onAttacked)="onAttackedHandler($event)"></event-child>`
})

export class EventComponent implements OnInit {
    damage: number = 0;

    onAttackedHandler(damage: number) {
        this.damage += damage;
    }

    ngOnInit() { }
}

@Component({
    selector: 'event-child',
    template: `
    <h1>The SkyWalker.</h1>
    <button (click)="attack()">Attack!</button><span>Damage: {{damage}} !</span>
    `
})
export class EventChildComponent {
    private baseDamage: number = 100;
    damage: number = 0;
    @Output() onAttacked = new EventEmitter<number>();

    attack() {
        this.damage = Math.random() * this.baseDamage;
        this.onAttacked.emit(this.damage);
    }
}


使用本地变量
在父组件模板中使用 #xxx 这样的本地变量绑定子组件，然后就可以在模板中直接使用子组件的属性作为数据源

@Component({
    selector: 'event',
    template: `
    <h1>I'm your father.</h1>
    <p>recived damage: {{damage}}</p>
    <p>last damage: {{child.damage}}</p>
    <event-child #child (onAttacked)="onAttackedHandler($event)"></event-child>`
})
export class EventComponent implements OnInit {
    damage: number = 0;

    onAttackedHandler(damage: number) {
        this.damage += damage;
    }

    ngOnInit() { }
}


使用 @ViewChild
父组件中使用 @ViewChild 获取子组件的引用，然后使用方法同上

import { Component, OnInit, Output, EventEmitter, ViewChild } from '@angular/core';

@Component({
    selector: 'event-child',
    template: `
    <h1>The SkyWalker.</h1>
    <button (click)="attack()">Attack!</button><span>Damage: {{damage}} !</span>
    `
})
export class EventChildComponent {
    private baseDamage: number = 100;
    damage: number = 0;
    @Output() onAttacked = new EventEmitter<number>();

    attack() {
        this.damage = Math.random() * this.baseDamage;
        this.onAttacked.emit(this.damage);
    }
}

// 注意，这里我交换了父子组件的位置，现在父组件的定义放在了下面

@Component({
    selector: 'event',
    template: `
    <h1>I'm your father.</h1>
    <p>recived damage: {{damage}}</p>
    <p>last damage: {{child.damage}}</p>
    <event-child  (onAttacked)="onAttackedHandler($event)"></event-child>`
})
export class EventComponent implements OnInit {
    damage: number = 0;

    @ViewChild(EventChildComponent) child: EventChildComponent;

    onAttackedHandler(damage: number) {
        this.damage += damage;
    }

    ngOnInit() { }
}
终极大招——组件间使用 service 通信
首先让我们在文件的开头定义一个 AttackService：

import { Component, OnInit, Injectable } from '@angular/core';
import { Subject } from 'rxjs/Subject';
import { Observable } from 'rxjs/Rx';


@Injectable()
export class AttackService {
    // 用来产生数据流的数据源
    private damageSource = new Subject<number>();
    // 把数据流转换成 Observable
    damage$ = this.damageSource.asObservable();

    attack(damage: number) {
        // 把伤害输入到数据流
        this.damageSource.next(damage);
    }
}
然后在父组件与子组件中订阅这个数据流：

@Component({
    selector: 'event-child',
    template: `
    <h1>The SkyWalker.</h1>
    <button (click)="attack()">Attack!</button><span>Damage: {{damage}} !</span>
    `
})
export class EventChildComponent {
    private baseDamage: number = 100;
    damage: number = 0;

    constructor(private attackService: AttackService) {
    }

    attack() {
        this.damage = Math.random() * this.baseDamage;
        // 天行者调用 AttackService 产生伤害
        this.attackService.attack(this.damage);
    }
}

// 注意，这里我交换了父子组件的位置，现在父组件的定义放在了下面

@Component({
    selector: 'event',
    providers: [AttackService], // 向父组件注入 AttackService，这样，父组件与子组件就能共享一个单例的 service
    template: `
    <h1>I'm your father.</h1>
    <p>recived damage: {{damage}}</p>
    <p>last damage: {{lastDamage}}</p>
    <event-child></event-child>`
})
export class EventComponent implements OnInit {
    damage: number = 0;
    lastDamage: number = 0;

    constructor(private attackService: AttackService) {
        // 父组件订阅来自天行者的伤害
        this.attackService.damage$.subscribe(damage => {
            this.lastDamage = damage;
            this.damage += damage;
        }, error => {
            console.log('error: ' + error);
        });
    }

    ngOnInit() { }
}
效果：



如果我们把 service 注册到根模块，那么，就可以在整个 app 中共享数据啦~
