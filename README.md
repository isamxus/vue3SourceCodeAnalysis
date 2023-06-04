# Vue3源码分析(Vue版本为3.2.41)
>## 一、响应式核心
>>### 1.数据代理--Proxy
>>> ### effect和数据的关系：
>>> * 在梳理数据代理逻辑之前，首先要了解Vue3中一个关键词--effect，effect翻译过来的意思就是效应、影响，在Vue3中，effect影响着页面的交互逻辑，effect执行(产生效应)又会依赖于不同的数据，每一个数据又会作用于不同的effect，而Vue3就是使用ES6的Proxy类来建立起数据和effect之间的关联。
>>> * 这里先不管effect的具体原理和作用，后续会分析effect的具体原理。
>>> * Proxy对数据进行get访问拦截操作，当effect执行时会访问数据，这些数据同时也会知道当前是哪个effect在执行，并且通过一个容器记录数据和当前执行的effect的映射关系，表示数据被应用到了哪些effect中(effect收集)。
>>> * 什么是响应式数据，就是数据捕捉到变化时，随之自动触发某些逻辑，而在Vue3中相当于数据改变后触发effect的执行，这个过程是通过Proxy对数据进行set拦截实现的，当拦截到改变数据的操作时，会通知数据在get阶段收集到的effect依次执行。
>>> ### 数据代理逻辑梳理：
>>> * 对于数据代理的逻辑这里只梳理重要步骤和关键方法，其他细枝末节的逻辑大家可自行到源码查阅或者我会在重要逻辑梳理完毕后挑选一些补充。
>>> * 在对数据进行get拦截操作时，会执行一个关键方法track，该方法主要是设置被代理对象中每个属性(即数据)对应的effect集合，当访问到某个具体属性时，获取对应的effect集合并传入下一个关键方法trackEffects中，下面是简化逻辑后的的代码：
```typescript
        const targetMap = new WeakMap(); 
        export function track(target: object, key: unknown) {
            let depsMap = targetMap.get(target)
            if (!depsMap) { 
                targetMap.set(target, (depsMap = new Map())) // 对于每一个被代理对象，针对该对象中每一个具体属性都会创建属性与effect集合的映射容器
            }
            let dep = depsMap.get(key) // 当访问到被代理对象中具体的属性时，获取属性对应的effect集合
            if (!dep) { 
                depsMap.set(key, (dep = new Set())) // 如果没有获取到effect集合，就创建该属性对应的effect集合
            }
            trackEffects(dep) // 将获取到的effect集合传入
        }
```
>>> * effect执行，如果访问到某个代理对象的具体属性，trackEffects方法就会将当前执行的effect加入到该属性对应的effect集合里，此时完成了数据对effect的收集过程，下面是简化逻辑后的的代码：
```typescript
        export function trackEffects(dep: Dep) {
            dep.add(activeEffect!) // activeEffect指的就是当前运行中的effect，dep就是track方法中传入的集合
        }
```
>>> * 此时的数据可以被称为是响应式的，因为数据已经建立了和不同effect的联系，当数据发生变更，Proxy的set拦截操作会触发关键方法trigger，该方法作用是，当更新某个代理对象的属性时，去除该属性之前在get阶段收集到的effect通过方法triggerEffects触发，下面是简化逻辑后的的代码：
```typescript
        export function trigger(target: object, key: unknown) {
            const depsMap = targetMap.get(target) 
            let deps = [];
            deps.push(depsMap.get(key)); // 取出属性通过get拦截收集到的effect
            triggerEffects(deps) // 执行effect
        }
```
>>> * triggerEffects简化逻辑后的的代码：
```typescript
        export function triggerEffects(deps) {
            for (const effect of deps) {
                effect.run(); // 依次执行数据收集到的effect
            }
        }
```
>>> * 至此，设置响应式数据的大致逻辑已经梳理完，下面是该逻辑粗略的流程图：
![响应式数据建立流程](https://github.com/isamxus/vue3SourceCodeAnalysis/blob/dd67c1872f22e5cac7a112965236d4f8aa99ca88/dataProxy.png)
>>> ### 部分分支逻辑：
>>> * track方法中，数据只有在当前用effect正在执行时，才会和effect建立映射关系：
```typescript
        export function triggerEffects(deps) {
            if (shouldTrack && activeEffect) { // activeEffect指的就是当前运行中的effect
                /** effect收集逻辑... */
            }
        }
```
>>> * 当数据更新，收集到的effect再次运行时，数据不会再对该effect进行收集，因为effect已经存在于该数据的effect集合里：
```typescript
        export function trackEffects(dep) {
            let shouldTrack = false;
            shouldTrack = !dep.has(activeEffect!) // 已经再集合中存在执行中的effect，所以不会再走下面判断体
            if (shouldTrack) {
                dep.add(activeEffect!)
            }
        }
```
>>### 2.reactive的实现原理
>>> * reactive是Vue3中常用的API之一，主要是将复杂引用类型的值(对象，数组等)进行Proxy代理，并使其具备响应式特性，下面是简化逻辑后的reactive方法：
```typescript
        export function reactive(target: object) {
            return createReactiveObject(
                target, // 要进行Proxy代理的对象
                baseHandlers // 该参数定义了Proxy的get和set拦截钩子
            )
        }
```
>>> * 在前文已经提到过Proxy代理的大致流程，在源码中，createReactiveObject方法就是进行代理的核心方法，代理后的对象会被存进一个Map容器里，如果要进行reactive的对象已经被代理过的并且代理对象已经在容器中，则直接返回该代理对象，下面是简化逻辑后的代码：
```typescript
        const proxyMap = new Map();
        export function createReactiveObject(target: object, baseHandlers:ProxyHandler<any>) {
            const existingProxy = proxyMap.get(target)
            if (existingProxy) {
                return existingProxy
            }
            const proxy = new Proxy(
                target,
                baseHandlers
            )
            proxyMap.set(target, proxy) // 缓存代理对象
            return proxy;
        }
```
>>> * 对于reactive来说，这里相比较Vue2的Object.defineProperty对于复杂嵌套对象的处理，Vue3做了一点优化，在触发拦截钩子get的时候，只有访问到嵌套对象里具体的对象属性的时候，才会对该对象进行Proxy代理，而在Vue2中，复杂对象会一直递归处理，直到对象内所有的对象属性都设置为响应式，下面是get钩子方法的简化逻辑：
```typescript
        new Proxy(target, {
            get(target, key){
                const res = Reflect.get(target, key, receiver) // 访问原始对象的属性，得到属性值
                track(target, key) // 开始收集effect，大致流程前文a已经梳理过
                if (isObject(res)) { // 如果属性值是对象，才会对该对象进行代理，而不是一开始就对原始对象进行递归代理
                    return reactive(res)
                }
                return res;
            }
        })
```
>>> * 同样，通过reactive代理的对象，在对象的属性值发生变化时，会触发set钩子方法，依次执行收集过的effect，下面是set钩子方法的简化逻辑：
```typescript
        new Proxy(target, {
            set(target, key, value){
                const result = Reflect.set(target, key, value, receiver) // 设置原始对象的属性值
                trigger(target, key, value) // 收集的effect依次执行，大致流程前文已经梳理过
                return result;
            }
        })
```
>>> * reactive只是Vue3中其中一个进行数据代理的API,接下来将会梳理更多类似的API以及响应式核心的其他分支逻辑，下面是reactive实现的大致流程图：
![reactive实现流程](https://github.com/isamxus/vue3SourceCodeAnalysis/blob/30504944abd2ff5f496ec6091ead5b456cc712ca/reactive.png)
>>### 3.ref的实现原理
>>> * ref同样是Vue3中常用的API之一，它主要用于基础类型的值的数据代理，但它的处理与reactive有所不同，下面是ref方法的简化逻辑：
```typescript
        export function ref(value?: unknown) {
            return new RefImpl(value);
        }
```
>>> * 从上面代码看到，ref的数据代理逻辑是通过核心类RefImpl来实现的，它改写了RefImpl实例属性value的getter，setter方法，只有通过.value来访问和修改属性类型的值，通过ref方法代理的数据才会具备响应式逻辑，ref并不会像reactive那样，对原始数据的代理对象进行缓存。
>>> * 当访问value属性时会触发trackEffect的effect收集流程，同样，改变属性value的值时，会触发triggerEffect的effect运行流程，trackEffect流程和triggerEffect在前文已经梳理过，下面是RefImpl类的简化逻辑：
```typescript
        class RefImpl<T> {
            private _value: T
            private _rawValue: T
            public dep?: Dep = new Set() // 建立ref实例与effect的映射，即收集effect
            constructor(value: T) {
                this._rawValue = value; // 记录原始的值，以便于出发setter方法时进行比较
                this._value = typeof value === 'object' ? reactive(value) : value; // 传给ref的参数可以是复杂引用类型的值，这个值会使用reactive方法进行响应式处理并且将返回的代理对象赋值给_value，同样可以通过.value来访问这个代理对象
            }
            get value() {
                trackEffect(this.dep) // 收集effect的流程，不再赘述
                return this._value
            }
            set value(newVal) {
                if (hasChanged(newVal, this._rawValue)) { // 比较新旧值，有变化才走下面逻辑
                    this._rawValue = newVal 
                    this._value = typeof value === 'object' ? reactive(value) : value;
                    triggerEffect(this.dep) // 触发effect的流程，不再赘述
                }
            }
        }
```
>>> * 根据我的使用经验，除了基础类型的值适合使用ref外，有一种情况同样适合使用ref，代码如下：
```typescript
        let proxy = reactive({}); // 如果有effect使用到proxy访问原始的对象，由于数据代理只会代理原始对象内的属性，而原始对象本身并不会被代理，所以并不会对运行中的effect进行收集
        proxy = {}; // 由于没有收集effect，改变proxy的值并不会触发effect的运行
```
>>> * 那么，如果我们想proxy也参与到收集和触发effect的逻辑中，可以这样做：
```typescript
        let proxy = ref({}); // 如果有effect使用到proxy，则要通过proxy.value访问原始对象，根据前文RefImpl类中value属性的getter方法逻辑，将会触发trackEffect的流程收集当前运行的effect
        proxy.value = {}; // 改变proxy.value的值，会依次执行收集到的effect
```
>>> * ref并不通过Proxy代理，它通过RefImpl类的处理通过属性value代理到原始数据上，这也是与reactive不同的地方，下面是ref的大致流程图：
![ref实现流程](https://github.com/isamxus/vue3SourceCodeAnalysis/blob/c7785df02e775837adc18201e3babd68fd5e23d1/ref.png)
>>### 4.effect的实现原理
>>> * effect是Vue3中非常重要的一个API，无论是组件渲染和更新，还是自定义watch，computed的运作都离不开effect，如果看完前文大家对effect可能会有点概念了，它就是贯穿整个Vue3运作的API，effect方法第一个参数接收一个回调函数，该函数在响应式数据更新的时候会执行，下面是effect方法的简化逻辑：
```typescript
        export function effect<T = any>(fn: () => T){
            const _effect = new ReactiveEffect(fn)
            _effect.run()
        }
```
>>> * 从上面代码可以看到，effect的具体逻辑都封装在ReactiveEffect核心类中，这个类在实例化后，会马上执行实例的run方法(watch和computed等API情况除外)，这里先说一个很关键的变量activeEffect，这个变量用来记录当前正在运行的effect，这样响应式数据就能够知道当前运行中的effect是哪个，从而进行effect的收集，下面ReactiveEffect类的简化逻辑：
```typescript
        export let activeEffect = null; // 该变量记录当前哪个effect在运行
        export class ReactiveEffect<T = any> {
            constructor(public fn: () => T){} // fn就是我们使用effect时传入的回调函数
            run(){
                activeEffect = this // 将activeEffect指向当前ReactiveEffect实例
                this.fn() // 执行回调，在回调中如果访问到响应式数据，那么这些响应式数据就会建立起和当前ReactiveEffect实例的联系
                activeEffect = null // 执行完回调后，将activeEffect置空
            }
        }
```
>>> * 看完上面代码，前文提到过effect执行，就是执行effect.run方法触发回调函数的运行，那么回调函数运行时，如果访问到响应式数据，就会触发trackEffects方法，每个响应式数据的dep就会收集这个运行中的effect，回顾一下trackEffects方法：
```typescript
        export function trackEffects(dep: Dep) {
            dep.add(activeEffect!) // activeEffect指的就是当前运行中的effect，dep就是track方法中传入的集合
        }
```
>>> * 当响应式数据发生变化时，则通知dep收集到的effect运行，这里的effect就是ReactiveEffect实例，它会调用实例的run方法完成数据变更后的逻辑，回顾一下triggerEffects方法：
```typescript
        export function triggerEffects(deps) {
            for (const effect of deps) {
                effect.run(); // 依次执行数据收集到的effect
            }
        }
```
>>> * 至此，对effect的作用有了一个大致的了解，当然effect方法和ReactiveEffect类还有很多分支逻辑，在后面的响应式API中还会进行渗透，梳理，下面是effect的大致流程图：
![effect实现流程](https://github.com/isamxus/vue3SourceCodeAnalysis/blob/4b9fd490efbf9255116371ee86f8681d78cf81a1/effect.png)
>>### 5.computed的实现原理
>>> * computed的实现和前文提到的ReactiveEffect息息相关，在梳理ReactiveEffect类的时候，构造函数的参数省略了第二个参数，这个参数是一个scheduler调度器，这个调度器在响应式数据更新的时候会运行，而不是执行ReactiveEffect实例的run方法。
```typescript
        export class ReactiveEffect<T = any> {
            constructor(
                public fn: () => T,
                public scheduler: EffectScheduler | null = null, // 调度器
            ) {
                /**... */
            }
        }
```
>>> * computed方法的第一个参数可以是一个回调函数，也可是一个带有get，set方法的对象，下面是简化后的逻辑：
```typescript
        export function computed<T>(getterOrOptions: ComputedGetter<T> | WritableComputedOptions<T>){
            let getter: ComputedGetter<T>
            let setter: ComputedSetter<T>
            typeof getterOrOptions === 'function' 
                ? (getter = getterOrOptions, setter = () => {}) // 如果传入参数类型是函数
                : (getter = getterOrOptions.get, setter = getterOrOptions.set); // 如果传入参数类型是带有get，set方法的对象
            const cRef = new ComputedRefImpl(getter, setter)
             return cRef;
        }
```
>>> * 从上面代码可以看到，computed处理产生getter，setter两个函数作为参数传入到核心类ComputedRefImpl构造函数，返回ComputedRefImpl实例，通过computed定义的数据依赖于getter函数的返回值，需要通过.value来访问。
>>> * ComputedRefImpl类内部通过定义value属性的get，set方法来完成响应式逻辑，这与ref实现类似，ComputedRefImpl实例在构造时会调用前文提到的ReactiveEffect类产生一个effect实例，传入ReactiveEffect实例的第一个参数是getter函数，重点是第二个参数，它是一个调度器，下面是ComputedRefImpl类的简化逻辑：
```typescript
        export class ComputedRefImpl<T> {
            private _value!: T // 记录getter函数执行后的返回值
            public readonly effect: ReactiveEffect<T> // 与ReactiveEffect实例关联
            public dep?: Dep = [] // 收集使用到这个computed数据的effect
            public _dirty = true // getter函数在函数依赖的响应式数据未更新时只会计算一次值
            constructor(getter: ComputedGetter<T>, private readonly _setter: ComputedSetter<T>) {
                this.effect = new ReactiveEffect(getter, () => { // ComputedRefImpl与ReactiveEffect实例关联起来
                    if (!this._dirty) {
                        this._dirty = true
                        triggerEffects(this.dep)
                    }
                })
            }
            get value() {
                trackEffects(this.dep) // 当访问这个computed定义的数据时，这个数据将收集当前正在运行的effect
                if (this._dirty) {
                    this._dirty = false
                    this._value = this.effect.run() // 计算getter函数的返回值，此时如果getter函数依赖其他响应式数据，那么其他响应式数据的dep就会收集这个ComputedRefImpl实例关联的effect
                }
                return this._value // 将getter计算后的值返回
            }
            set value(newValue: T) {
                this._setter(newValue)
            }
        }
```
>>> * 当我们通过.value访问computed定义的数据时，这个数据会先收集当前正在运行的effect，然后才触发getter函数的执行。
>>> * getter函数内可能要访问多个响应式数据才能计算它的返回值，当访问这些响应式数据时，这些响应式数据的dep会收集computed方法产生ComputedRefImpl实例关联的effect。
>>> * 第一次访问computed定义的数据，getter函数会执行一次，但是getter函数内依赖的多个响应式数据在没有更新的情况下，无论后续多少次访问computed定义的数据，getter都不会执行，访问到的都是第一次计算后的值，这有利于性能提升。
>>> * 当getter函数依赖的响应式数据更新时，这些数据的dep收集的effect(computedRefImpl实例关联)，通过triggerEffect执行，但这个effect不是执行run方法，而是执行ReactiveEffect实例创建的时传入的调度器：
```typescript
        function triggerEffect(effect: ReactiveEffect) {
            if (effect.scheduler) { // 前文省略了这个逻辑
                effect.scheduler()
            } else {
                effect.run()
            }
        }
```
>>> * 来看看这个调度器做了什么：
```typescript
        () => {
            if (!this._dirty) { 
                this._dirty = true // 第一次计算getter函数的返回值会将_dirty置为false，此时由于getter函数依赖的数据改变，这里要给它置为true，下一次访问computed定义的数据时才会重新计算getter函数的返回值
                triggerEffects(this.dep) // 通知compted定义的数据收集的effect执行
            }
        }
```
>>> * 下面是computed实现的大致流程图：
![computed实现流程](https://github.com/isamxus/vue3SourceCodeAnalysis/blob/55ec895ad71998b3edaf07d29d45fad768c751fb/computed.png)
>>### 6.watch的实现原理
>>> * watch用于监听数据的变化，并在数据变化后执行某些逻辑，watch的实现也与ReactiveEffect类相关，跟computed一样，在创建ReactiveEffect类实例时，同样会传入一个调度器。
>>> * watch的第一个参数是需要监听的响应式数据，在watch内部会处理成函数调用的形式返回这个数据，watch的第二个参数是的监听的数据变化后的回调，这个回调默认情况下是异步触发的，watch的第三个参数是一个配置对象，可以指定回调的执行时机，监听的数据是否深度监听等等。
>>> * 在watch创建时会把传入的要监听的数据包装成一个getter函数，并创建一个effect，默认执行一次effect.run方法，effect.run方法执行的就是这个getter，getter函数会访问具体的响应式数据，并当作旧值储存，此时响应式数据的dep就会收集这个watch方法创建的effect。
>>> * 在要监听的响应式数据更新时，收集到的effect会执行创建ReactiveEffect类实例时传入的调度器，这个调度器内部会再执行一次getter方法拿到最新的数据，然后默认会异步执行watch方法传入的回调，将新值和旧值作为参数传入回调中。
>>> * 下面是watch的简化逻辑：
```typescript
        export function watch(getter, cb){
            const scheduler = () => {
                const newValue = effect.run(); // 执行getter，拿到新值
                if (oldValue === newValue) return; // 旧值和新值一样就不会触发回调
                Promise.resolve().then(() => cb(newValue, oldValue)) // 异步执行watch方法的传入的回调
                oldValue = newValue; /// 更新旧值，下次响应式数据更新时与新值比较
            }
            const effect = new ReactiveEffect(getter, scheduler) // 创建一个effect，将getter和调度器scheduler传入
            let oldValue = effect.run() // 先执行一次getter，拿到要监听的数据，作为旧值
        }
```
>>> * 从上面代码可以看出，watch的实现逻辑比computed稍微简单一点，就是在创建effect的时候，马上执行run方法通过getter函数访问响应式数据，响应式数据收集effect，当响应式数据更新，执行effect的调度器，调度器内部实际上就是执行传入的回调。
>>> * 下面是watch方法实现的大致流程图：
![watch实现流程](https://github.com/isamxus/vue3SourceCodeAnalysis/blob/573b5626a417543ac4cbee14d6ff61c301cb5ddd/watch.png)
