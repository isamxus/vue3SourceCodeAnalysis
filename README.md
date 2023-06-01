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
>>> * reactive只是Vue3中其中一个进行数据代理的API,接下来将会梳理更多类似的API以及响应式核心的其他分支逻辑。