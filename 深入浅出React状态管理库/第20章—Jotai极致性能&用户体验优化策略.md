本文会围绕 jotai-scheduler 库进行讲解，jotai-scheduler 库项目地址：https://github.com/jotaijs/jotai-scheduler 可以给该项目点一个 Star 🌟 方便项目中使用时可以快速找到该优化方案。

# 传统 Jotai 或者其它状态管理库有什么问题？

假设我们现在有这样一个页面：


<p align=center><img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/39aca2399589494583b31eb57d062e56~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=847&h=779&s=14573&e=png&b=ffffff" alt="demo.png" width="70%" /></p>

它包含了一个应用中通用的元素 —— Header、Footer、Sidebar、Content。每一个元素代表了一个组件，在这些组件中可能会复用同一个全局状态。

对应代码可能是这样的：


```js
const anAtom = atom(0);

const Header = () => {
  const num = useAtomValue(anAtom);
  return <div className="header">Header-{num}</div>;
};

const Footer = () => {
  const num = useAtomValue(anAtom);
  return <div className="footer">Footer-{num}</div>;
};

const Sidebar = () => {
  const num = useAtomValue(anAtom);
  return <div className="sidebar">Sidebar-{num}</div>;
};

const Content = () => {
  const [num, setNum] = useAtom(anAtom);
  return (
    <div className="content">
      <div>Content-{num}</div>
      <button onClick={() => setNum((num) => ++num)}>+1</button>
    </div>
  );
};
```

即当我们点击按钮时会更新全局状态，并触发所有的组件 re-render，为了模拟更真实的场景，我们手动来模拟繁重的渲染计算过程：


```js
const simulateHeavyRender = () => {
  const start = performance.now();
  while (performance.now() - start < 500) {}
};
```

并在组件中进行调用：

```js
const anAtom = atom(0);

const Header = () => {
  simulateHeavyRender();
  const num = useAtomValue(anAtom);
  return <div className="header">Header-{num}</div>;
};

const Footer = () => {
  simulateHeavyRender();
  const num = useAtomValue(anAtom);
  return <div className="footer">Footer-{num}</div>;
};

const Sidebar = () => {
  simulateHeavyRender();
  const num = useAtomValue(anAtom);
  return <div className="sidebar">Sidebar-{num}</div>;
};

const Content = () => {
  simulateHeavyRender();
  const [num, setNum] = useAtom(anAtom);
  return (
    <div className="content">
      <div>Content-{num}</div>
      <button onClick={() => setNum((num) => ++num)}>+1</button>
    </div>
  );
};
```

也就是说在渲染每一个组件时都会包含了 500ms 的延迟，这时候看看当我们点击按钮时的效果：


<p align=center><img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7372aa5d6f6b43e3902aac69e4ab634c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=840&h=776&s=173253&e=gif&f=193&b=fdfcff" alt="before-optimization.gif" width="70%" /></p>

可以看到，当我们点击按钮时，需要等较长的时间才能看到状态的变化，并且 `<Header />`、`<Footer />`、`<Sidebar />`、`<Content />` 引用的状态同一时间发生了变化。

对应打开 Chrome Performance：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3a711a2c34bf4a8b9ec3e0fb57006300~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2120&h=928&s=137997&e=png&b=faf5f4)

可以看到此时包含了 3 个 Long Task，对应点击三次按钮，这无疑对于用户体验是非常差的。

可能现在有的小伙伴会说 React18 不是有并发更新吗？为什么不开启并发更新来解决这个问题！

好！现在我们使用 `useTransition` 来开启并发更新：


```js
const Content = () => {
  simulateHeavyRender();
  const [num, setNum] = useAtom(anAtom);
  const [isPending, startTransition] = useTransition();
  return (
    <div className="content">
      <div>Content-{num}</div>
      <button
        onClick={() => {
          startTransition(() => {
            setNum((num) => ++num);
          });
        }}
      >
        +1
      </button>
    </div>
  );
};
```

此时效果：


<p align=center><img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/134eb6b57f08457896b44673ea9d4610~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1416&h=1464&s=272336&e=gif&f=277&b=fdfcff" alt="20240822221114_rec_.gif" width="70%" /></p>


对应 Chrome Performance：


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4882a07b4291457aa346106460207206~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2034&h=836&s=138593&e=png&b=f8f2f0)

可以看到，即使开启了并发更新，用户仍然需要等待同样的时间来看到状态的更新，和上面的区别仅仅是一个长任务被拆分为了数个子渲染任务。由于每个组件渲染时间是固定的，因此无论何种方式都无法减少总的渲染时长，我们唯一能做的，就是让每个组件渲染完立刻呈现给用户，而区分组件渲染先后顺序我们称之为“渲染优先级”。

通常来说用户更关心内容区域，因此我们可以假设在现在的页面中渲染优先级为：`Content > Sidebar > Header = Footer`，也就是说我们应该先把 `<Content />` 组件渲染出来之后立即呈现给用户，这样就大大提前关键内容展示给用户的时机，带来更好的性能以及用户体验！

那我们怎么能做到这一点呢？答案就是 jotai-scheduler。


# 基于 jotai-scheduler 库进行优化

jotai-scheduler API 和 原生 Jotai 极为相似，对应关系为：

- `useAtom` --> `useAtomWithSchedule`
- `useAtomValue` --> `useAtomValueWithSchedule`
- `useSetAtom` --> `useSetAtomWithSchedule`

使用上唯一和 Jotai 的一点区别是可以额外传入 `priority` 代表渲染优先级：


```js
import { LowPriority, useAtomValueWithSchedule } from 'jotai-scheduler'

const [num, setNum] = useAtomWithScheduleanAtom, {
  priority: LowPriority,
});

const num = useAtomValueWithSchedule(anAtom, {
  priority: LowPriority,
});
```

`priority` 可以传入三个值 —— `ImmediatePriority`, `NormalPriority`, `LowPriority`。`priority` 是一个可选项，如果不传入默认为 `NormalPriority`，此时的行为等同于原生 Jotai API。因此，**现在你完全可以使用这些 API 替换掉原生 Jotai API**，即使你不使用渲染优先级的能力。

好，现在让我们来看一下效果：


```js
const anAtom = atom(0);

const Header = () => {
  const num = useAtomValueWithSchedule(anAtom, {
    priority: LowPriority,
  });
  return <div className="header">Header-{num}</div>;
};

const Footer = () => {
  const num = useAtomValueWithSchedule(anAtom, {
    priority: LowPriority,
  });
  return <div className="footer">Footer-{num}</div>;
};

const Sidebar = () => {
  const num = useAtomValueWithSchedule(anAtom);
  return <div className="sidebar">Sidebar-{num}</div>;
};

const Content = () => {
  const [num, setNum] = useAtomWithSchedule(anAtom, {
    priority: ImmediatePriority,
  });
  return (
    <div className="content">
      <div>Content-{num}</div>
      <button onClick={() => setNum((num) => ++num)}>+1</button>
    </div>
  );
};
```

此时效果：

<p align=center><img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f55a6f832b74cb58cd26accfb18f32b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=844&h=770&s=169549&e=gif&f=183&b=fdfcff" alt="after-optimization.gif" width="70%" /></p>

对应 Chrome Performance：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/16b98fdad0644b688d87c53731c046e8~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2024&h=886&s=116738&e=png&b=f6f8ff)

可以看到，重要的内容更早的展现给了用户（时间缩短 75%），这也意味着带来了更好的用户体验。


# 更多案例

再举一个通用的实际场景，我们经常会看到瀑布流布局的页面，并且当我们点击某个 Item 时候会展开这个卡片的详情：


![20240823220123_rec_.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bad10fb969b2440a92c2826ff3d5d110~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1902&h=1090&s=13703998&e=gif&f=87&b=fcfbfe)

我们对展开的卡片做一些操作其实对应也会反映在 Item 上，比如当我点赞的时候，退出展开的卡片后在 Item 上也可以看到点赞的效果：


![20240823220440_rec_.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/08319fc5c54943d8b78e53632ed6b377~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1902&h=1090&s=10119902&e=gif&f=64&b=fbfafe)

也就是说，这些信息的状态是共享的，当状态发生变化时需要触发组件 re-render，但是 Item 在卡片下面，因此我们无需立刻 re-render 下层的组件，也就是说上层的渲染优先级 > 下层的元素。或者当我们有非常多的 Item 时，屏幕内的元素优先级 > 屏幕外的元素。

我们可以模拟一下这种情况，点击查看 Demo：https://codesandbox.io/p/sandbox/jotai-scheduler-demo2-qryl38?file=%2Fsrc%2FApp.js%3A56%2C18

首先我们写一个 Hook 借助 [IntersectionObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/IntersectionObserver) API 用来判断组件是否位于屏幕内部：


```js
function useIsVisible() {
  const ref = useRef(null);
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        setIsVisible(entry.isIntersecting);
      }
    );

    if (ref.current) {
      observer.observe(ref.current);
    }

    return () => {
      if (ref.current) {
        observer.unobserve(ref.current);
      }
    };
  }, [ref]);

  return [ref, isVisible];
}
```

最后我们来编写一下 `<App />` 组件与 `<Item />` 组件：


```js
const Item = () => {
  simulateHeavyRender();
  const [ref, isVisible] = useIsVisible();
  const [num, setNum] = useAtom(anAtom);
  return (
    <div
      ref={ref}
      style={{
        height: "50px",
        width: "50%",
        margin: "10px",
        textAlign: "center",
        border: "2px solid black",
      }}
    >
      <div>
        {num} {isVisible ? "visible" : "not visible"}
      </div>
      <button
        onClick={() => {
          setNum(num + 1);
        }}
      >
        +1
      </button>
    </div>
  );
};

const App = () => {
  const items = new Array(100).fill(0);

  return (
    <div
      style={{
        display: "flex",
        flexDirection: "column",
        alignItems: "center",
      }}
    >
      {items.map(() => (
        <Item />
      ))}
    </div>
  );
};
```

同样为了模拟真实的场景给每个 `<Item />` 组件增加了渲染延迟，同时每个 `<Item />` 组件都引用了同一个全局状态，并且当点击按钮时会更新这个状态：


<p align=center><img src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/708265d3dcd84ded845b40089f55fb13~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1474&h=1140&s=68134&e=png&b=ffffff" alt="image.png" width="90%" /></p>

在原先我们使用 Jotai API `useAtomValue` 时，当我们点击按钮时效果：


<p align=center><img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4c5c0466f88c4909bc15048af5a08a3c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1488&h=1128&s=182439&e=gif&f=165&b=fdfcff" alt="20240823223936_rec_.gif" width="90%" /></p>

可以看到非常的卡顿，这是因为 React 需要将全部 Item 都渲染出来。按照我们前面的理论，在屏幕内的组件应该是更重要的，并且我们可以通过 `useIsVisible` Hook 判断出哪个组件位于屏幕内，从而赋予不同的优先级，从而优先去渲染屏幕内的组件：


```js
const [num, setNum] = useAtomWithSchedule(anAtom, {
  priority: isVisible ? ImmediatePriority : NormalPriority,
});
```

最终效果：


<p align=center><img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/584d6290eac44ae5895d89478e2e023c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1476&h=1132&s=58711&e=gif&f=35&b=fdfcff" alt="20240823224306_rec_.gif" width="90%" /></p>

可以看到非常丝滑！

# 总结

本文围绕了 jotai-scheduler 库进行了讲解，使用 jotai-scheduler 可以区分渲染优先级，从而将重要的组件优先渲染出来，这是直接使用 Jotai 所做不到的，jotai-scheduler 库项目地址：https://github.com/jotaijs/jotai-scheduler 可以给该项目点一个 Star 🌟 方便项目中使用时可以快速找到该优化方案。

至此我们完成了全部 Jotai 内容的介绍。