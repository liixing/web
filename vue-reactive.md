# vue 3.0数据响应原理

<a name="SMKTf"></a>
## 数据响应式原理

vue 3.0通过 proxy 实现数据的双向绑定。关于 proxy，具有强大的代理功能，首先我们来看一下 proxy 的 get 和set，以及一些比较容易忽视的细节。

<a name="S4193"></a>
#### 默认行为

```javascript
    const data = { foo: "foo" };
    const p = new Proxy(data, {
      get(target, key, receiver) {
        return target[key];
      },
      set(target, key, value, receiver) {
        console.log("set value");
         target[key] = value;
      }
    });

    p.foo = 123;
  // set value
```

p 通过 proxy 代理了原始数据，当对 p 进行操作时，就可以侦测到数据变化。但这么写是有问题的，当代理对象是数组时，会报错。

```javascript
    const data = [1, 2, 3];
    const p = new Proxy(data, {
      get(target, key, receiver) {
        return target[key];
      },
      set(target, key, value, receiver) {
        console.log("set value");
        target[key] = value;
      }
    });

    p.push(4);
// Uncaught TypeError: 'set' on proxy: trap returned falsish for property '3'
    at Proxy.push
```

因为在代理对象为数组时，push 操作不仅只改变了当前数据，还引发了数组索引的改变，所以要将代码改为

```javascript
    const data = [1, 2, 3];
    const p = new Proxy(data, {
      get(target, key, receiver) {
        console.log("get value:", key);
        return target[key];
      },
      set(target, key, value, receiver) {
        console.log("set value:", key, value);
        target[key] = value;
        return true;
      }
    });

    p.push(4);
// get value: push
   get value: length
   set value: 3 4
   set value: length 4
```

可以看到，当我们进行 push 操作时，不仅将数组的下标的第三位的值设置成了4，还将数组的 length 也设置成了 4 。同时，这个操作还触发 get 去获取 push 和 length 两个属性。我们可以通过 Reflect 来返回相应的默认行为，来处理一些相对复杂的默认行为

```javascript
    const data = [1, 2, 3];
    const p = new Proxy(data, {
      get(target, key, receiver) {
        console.log("get value:", key);
        return Reflect.get(target, key, receiver);
      },
      set(target, key, value, receiver) {
        console.log("set value:", key, value);
        return Reflect.set(target, key, value, receiver);
      }
    });

    p.push(4);
// get value: push
   get value: length
   set value: 3 4
   set value: length 4
```

<a name="KpjA1"></a>
#### 多次触发 get 和 set
从前面的例子可以看出，当代理对象是数组，当我们进行操作时，会多次触发 get 和 set,我们再来看一个例子

```javascript
    const data = [1, 2, 3];
    const p = new Proxy(data, {
      get(target, key, receiver) {
        console.log("get value:", key);
        return Reflect.get(target, key, receiver);
      },
      set(target, key, value, receiver) {
        console.log("set value:", key, value);
        return Reflect.set(target, key, value, receiver);
      }
    });

    p.unshift(4);
    // get value: unshift
       get value: length
       get value: 2
       set value: 3 3
       get value: 1
       set value: 2 2
       get value: 0
       set value: 1 1
       set value: 0 4
       set value: length 4
```

当我们对数组进行 unshift 操作时，首先拿到数组最末位下标 2，然后开辟一个新下标3存放原有的数组末位数值3，然后将数组前两位向后挪，将索引为 0 的位置放入 unshft 的值 4，由此引发了多次 set 操作。显然，如果我们触发了多次 set 和 get ,数据的响应式系统就要触发多次，首先我们用自己的思路来解决这个问题。

```javascript
    const reactive = (data, cb) => {
      let timer = null;
      return new Proxy(data, {
        get(target, key, receiver) {
          return Reflect.get(target, key, receiver);
        },
        set(target, key, value, receiver) {
          clearTimeout(timer);
          timer = setTimeout(() => {
            cb && cb();
          }, 0);
          return Reflect.set(target, key, value, receiver);
        }
      });
    };

    const arr = [1, 2];
    const p = reactive(arr, () => {
      console.log("trigger");
    });
    p.push(3);
   //trigger
```

这里我们使用定时器，使用类似防抖的方式解决了多次触发响应的问题。 可以看到，最后只输出了一次 trigger。

<a name="4UGUV"></a>
#### 代理深度问题
我们上面代理的对象都只有一层，当代理对象的结构有多层时

```javascript
    const reactive = (data, cb) => {
      let timer = null;
      return new Proxy(data, {
        get(target, key, receiver) {
          return Reflect.get(target, key, receiver);
        },
        set(target, key, value, receiver) {
          clearTimeout(timer);
          timer = setTimeout(() => {
            cb && cb();
          }, 0);
          return Reflect.set(target, key, value, receiver);
        }
      });
    };

    const data = { bar: { key: 1 } };
    const p = reactive(data, () => {
      console.log("trigger");
    });
    p.bar.key = 3;
```

发现 trigger并没有被打印，说明 proxy 代理的对象只能到第一层，对于对象内部的深度侦测，是要自己实现的。

```javascript
    const data = { a: { b: { c: 1 } } };
    const p = new Proxy(data, {
      get(target, key, receiver) {
        const res = Reflect.get(target, key, receiver);
        console.log(res);
        return res;
      },
      set(target, key, value, receiver) {
        return Reflect.set(target, key, value, receiver);
      }
    });

    p.a.b.c = 3;
```
  <br />我们打印 Reflect.get(target, key, receiver) 可以发现，当代理的对象是多层结构时， `Reflect.get` 会返回对象的内层结构 ，所以我们就可以使用递归来解决深度监听的问题。

```javascript
    const reactive = (data, cb) => {
      let res = null;
      let timer = null;
      res = Array.isArray(data) ? [] : {};
      for (let key in data) {
        if (typeof data[key] === "object") {
          res[key] = reactive(data[key], cb);
        } else {
          res[key] = data[key];
        }
      }
      return new Proxy(res, {
        get(target, key) {
          return Reflect.get(target, key);
        },
        set(target, key, value) {
          clearTimeout(timer);
          timer = setTimeout(() => {
            cb && cb();
          }, 0);
          return Reflect.set(target, key, value);
        }
      });
    };
    let data = { foo: "foo", bar: [1, 2] };
    let p = reactive(data, () => {
      console.log("trigger");
    });
    p.bar.push(3);
    // trigger
```

这个时候我们代理的对象 p 每层都带上了 proxy 标志。到这里我们用自己的方法解决了使用 proxy 实现数据监听的细节问题，但是我们的实现方式不够优雅，特别使用递归代理当数据对象比较大的时候将会是一个巨大性能隐患，我们来看一下 vue3.0的源码，看一下他是怎么实现的。

<a name="ZsvYI"></a>
#### vue3.0中的reactivity
vue3.0中实现响应式数据的核心在reactive.ts文件中，这里我将源码进行一定的简化

```javascript
function reactive(target) {
   return createReactiveObject(
    target,
    rawToReactive,
    reactiveToRaw,
    mutableHandlers,
    mutableCollectionHandlers
  )
}

function createReactiveObject(
  target: any,
  toProxy: WeakMap<any, any>,
  toRaw: WeakMap<any, any>,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>
) {
   let observed = toProxy.get(target)
  // 原数据已经有相应的可响应数据, 返回可响应数据
  if (observed !== void 0) {
    return observed
  }
  // 原数据已经是可响应数据
  if (toRaw.has(target)) {
    return target
  }
  //collectionTypes = new Set<Function>([Set, Map, WeakMap, WeakSet])
  const handlers = collectionTypes.has(target.constructor)
    ? collectionHandlers
    : baseHandlers
  //生成代理对象
  observed = new Proxy(target, handlers)
  toProxy.set(target, observed)
  toRaw.set(observed, target)
  if (!targetMap.has(target)) {
    targetMap.set(target, new Map())
  }
  return observed
}
```

从上面源码可以看出,根据 target 类型适配不同的 handler ，如果是集合 （`Map、Set`）就使用 collectionHandlers ，是其他类型就使用 baseHandlers。首先来看 baseHandlers ,在 baseHandlers.ts 文件中

```typescript
export const mutableHandlers: ProxyHandler<any> = {
  get: createGetter(),
  set,
  deleteProperty,
  has,
  ownKeys
}

function createGetter() {
  return function get(target: any, key: string | symbol, receiver: any) {
    const res = Reflect.get(target, key, receiver)
    return isObject(res) ? reactive(res):res
  }
}

 hasOwnProperty = Object.prototype.hasOwnProperty
 hasOwn = (
  val: object,
  key: string | symbol
): key is keyof typeof val => hasOwnProperty.call(val, key)


export const enum OperationTypes {
  SET = 'set',
  ADD = 'add',
  DELETE = 'delete',
  CLEAR = 'clear',
  GET = 'get',
  HAS = 'has',
  ITERATE = 'iterate'
}


function set(
  target: any,
  key: string | symbol,
  value: any,
  receiver: any
): boolean {
  value = toRaw(value)
  const hadKey = hasOwn(target, key)
  const oldValue = target[key]
  const result = Reflect.set(target, key, value, receiver)
  // don't trigger if target is something up in the prototype chain of original
  if (target === toRaw(receiver)) {
     if (!hadKey) {
        trigger(target, OperationTypes.ADD, key)
      } else if (value !== oldValue) {
        trigger(target, OperationTypes.SET, key)
      }
  }
  return result
}
```

在 baseHandlers 中，可以看到，判断了 Reflect 返回的数据是否还是对象，如果还是对象则再走一次 reactive ，从而实现了对象的深度监听。在 set 函数中，举个例子，当我们对数组进行 psuh 操作时，首先传进来新增的索引，hasOwn 返回 false ，此时执行 trigger 函数，然后传进数组的长度，此时 hasOwn 返回 true，然后再判断是否发生了变化，如果有变化执行 trigger 函数，没变化则不执行，避免了多次 trigger 的问题 

再来看 collectionHandlers 

```typescript
export const mutableCollectionHandlers: ProxyHandler<any> = {
  get: createInstrumentationGetter(mutableInstrumentations)
}

function createInstrumentationGetter(instrumentations: any) {
  return function getInstrumented(
    target: any,
    key: string | symbol,
    receiver: any
  ) {
    target =
      hasOwn(instrumentations, key) && key in target ? instrumentations : target
    return Reflect.get(target, key, receiver)
  }
}

const mutableInstrumentations: any = {
  get(key: any) {
    return get(this, key, toReactive)
  },
  get size() {
    return size(this)
  },
  has,
  add,
  set,
  delete: deleteEntry,
  clear,
  forEach: createForEach(false)
}
```

由于集合没有 set 方法，集合赋值是采用的 add 方法。这里新创建了一个和集合对象具有相同属性和方法的普通对象，在集合对象 get 操作时将 target 对象换成新创建的普通对象。这样，当调用 get 操作时 Reflect 反射到这个新对象上，当调用 set 方法时就直接调用新对象上可以触发响应的方法
