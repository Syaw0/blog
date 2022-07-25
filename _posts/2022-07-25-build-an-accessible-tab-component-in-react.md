---
layout: post
title: "Build an accessible Tab component in React"
date: 2022-07-25 18:21:10 +0900
categories: React
image: /static/a11y-tab.png
---

> You can watch my [Youtube video explanation](https://www.youtube.com/watch?v=QKlJNuckVcU) for this post.

So I am asked to implement Tab component in React, I thought it was easy but actually it is not, since we need to take Accesibility into consideration.

## The straightforward approach

Well, here is my first try, just define types for the tabs and get them rendered.

```ts
const tabs: Tab[] = [
  {
    label: "tab 1",
    key: "tab1",
    content: <p> content for tab 1</p>,
  },
  {
    label: "tab 2",
    key: "tab2",
    content: <p> content for tab 2</p>,
  },
  {
    label: "tab 3",
    key: "tab3",
    content: <p> content for tab 3</p>,
  },
];
export default function App() {
  return <Tabs tabs={tabs} defaultSelectedTab={tabs[0]} />;
}

type Tab = {
  label: string;
  key: string;
  content: React.ReactNode;
};

function Tabs({
  tabs,
  defaultSelectedTab,
}: {
  tabs: Tab[];
  defaultSelectedTab: Tab;
}) {
  const [selectedTab, selectTab] = useState(defaultSelectedTab);
  return (
    <div>
      <div>
        {tabs.map((tab) => {
          const style = { fontWeight: selectedTab === tab ? "bold" : "normal" };
          return (
            <button key={tab.key} onClick={() => selectTab(tab)} style={style}>
              {tab.label}
            </button>
          );
        })}
      </div>
      {selectedTab.content}
    </div>
  );
}
```

## Problem above approach

1. Not flexible enough, too strong assumptions about the data structure and html structure. We must pass in the data and contents through `Tab[]` and we are not easily to change html structure. Suppose we want to add a title in one Tab, and add some extra controls in another Tab, it won't go well

2. accessiblity issue, yeah it is not accessible.

## Solution - mininum abstraction

In my video about [Inversion of Control](https://www.youtube.com/watch?v=lxePHsJAJ8I), I've explained that the better abstraction is minimum abstraction.

So what is Tabs exactly? What state should it hold?

Think about it for a few seconds and we would soon realize that **The core of Tab system is that it holds the selected tab**

Yeah, that's it, the `selected tab`, nothing more. It doesn't care about the HTML structure nor the data structure.

So basically we want a Tabs component to hold the selected tab, and allow descendants to use selected tab and change it.

1. Tabs: the component to hold the state and also internal prefix
2. TabList: the component to hold elements of tabs, handls the keyboard navigation
3. Tab: the single tab component
4. TabPanel: the single tab panel component.

Our goal is to support something like this

```tsx
function App() {
  return (
    <Tabs defaultSelectedTab="tab2">
      <TabList aria-label="jser tabs">
        <Tab tab="tab1">tab 1</Tab>
        <Tab tab="tab2">tab 2</Tab>
        <Tab tab="tab3">tab 3</Tab>
      </TabList>
      <TabPanel tab="tab1">content for tab 1</TabPanel>
      <TabPanel tab="tab2">content for tab 2</TabPanel>
      <TabPanel tab="tab3">content for tab 3</TabPanel>
    </Tabs>
  );
}
```

We can see that all the logic are encapsulated in these separate components, we can insert extra titles as we want and there is no assumption on the data structures.

Perfect, below is the code

## A better approach - with accessibility built-in

You can find [the full code here](https://stackblitz.com/edit/react-ts-2bfx2c?file=App.tsx).

```ts
const TabsContext = React.createContext<{
  selectedTab: string | null;
  selectTab: (tab: string) => void;
  tabsPrefix: string;
}>({
  tabsPrefix: "",
  selectedTab: null,
  selectTab: (tab: string) => {
    throw new Error("should not be used without TabsContext.Provider");
  },
});

function Tabs({
  children,
  defaultSelectedTab,
}: {
  children: React.ReactNode;
  defaultSelectedTab: string;
}) {
  const tabsPrefix = React.useMemo(() => {
    // use some unique id generator
    // return uid()
    return "tabxxx";
  }, []);
  const [selectedTab, selectTab] = useState(defaultSelectedTab);

  const contextValue = React.useMemo(
    () => ({
      selectTab,
      selectedTab,
      tabsPrefix,
    }),
    [selectedTab, selectTab]
  );

  return (
    <TabsContext.Provider value={contextValue}>
      <div>{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({
  children,
  "aria-label": ariaLabel,
}: {
  children: React.ReactNode;
  "aria-label": string;
}) {
  const refList = React.useRef<HTMLDivElement>(null);

  const onKeyDown = useCallback((e: React.KeyboardEvent) => {
    const list = refList.current;
    if (!list) return;
    const tabs = Array.from<HTMLElement>(
      list.querySelectorAll('[role="tab"]:not([diabled])')
    );
    const index = tabs.indexOf(document.activeElement as HTMLElement);
    if (index < 0) return;

    switch (e.key) {
      case "ArrowUp":
      case "ArrowLeft": {
        const next = (index - 1 + tabs.length) % tabs.length;
        tabs[next]?.focus();
        break;
      }
      case "ArrowDown":
      case "ArrowRight": {
        const next = (index + 1 + tabs.length) % tabs.length;
        tabs[next]?.focus();
        break;
      }
    }
  }, []);
  return (
    <div
      ref={refList}
      role="tablist"
      aria-label={ariaLabel}
      onKeyDown={onKeyDown}
    >
      {children}
    </div>
  );
}

function Tab({ children, tab }: { tab: string; children: React.ReactNode }) {
  const { selectedTab, selectTab, tabsPrefix } = React.useContext(TabsContext);
  const style = { fontWeight: selectedTab === tab ? "bold" : "normal" };
  return (
    <button
      role="tab"
      aria-selected={selectedTab === tab}
      aria-controls={`tab-${tabsPrefix}-tabpanel-${tab}`}
      onClick={() => selectTab(tab)}
      tabIndex={selectedTab === tab ? 0 : -1}
      style={style}
    >
      {children}
    </button>
  );
}

function TabPanel({
  children,
  tab,
}: {
  tab: string;
  children: React.ReactNode;
}) {
  const { selectedTab, tabsPrefix } = React.useContext(TabsContext);

  if (selectedTab !== tab) return null;

  return (
    <div role="tabpanel" tabIndex={0} id={`tab-${tabsPrefix}-tabpanel-${tab}`}>
      {children}
    </div>
  );
}
```

The code is pretty straightforward, you can refer to my [Youtube video](https://www.youtube.com/watch?v=QKlJNuckVcU) for the whole thinking process.
