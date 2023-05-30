# Vue3源码分析
>## 一、响应式核心
>>### 1.数据代理--Proxy
>>> ### effect和数据的关系：
>>> * 在梳理数据代理逻辑之前，首先要了解Vue3中一个关键词--effect，effect翻译过来的意思就是效应、影响，在Vue3中，effect影响着页面的交互逻辑，effect执行(产生效应)又会依赖于不同的数据，每一个数据又会作用于不同的effect，而Vue3就是使用ES6的Proxy类来建立起数据和effect之间的关联。
>>> * 这里先不管effect的具体原理和作用，后续会分析effect的具体原理。
>>> * Proxy对数据进行get访问拦截操作，当effect执行时会访问数据，这些数据同时也会知道当前是哪个effect在执行，并且通过一个容器记录数据和当前执行的effect的映射关系，表示数据被应用到了哪些effect中(effect收集)。
>>> * 什么是响应式数据，就是数据捕捉到变化时，随之自动触发某些逻辑，而在Vue3中，相当于数据改变后触发effect的执行，这个过程是通过Proxy对数据进行set拦截实现的，当拦截到改变数据的操作时，会通知数据在get阶段收集到的effect依次执行。
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