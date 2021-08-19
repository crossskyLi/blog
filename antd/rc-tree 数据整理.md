
# antd 基础组件 React Component RC-Tree 


## 源码
[rc-tree](https://github.com/react-component/tree)

src/Tree.tsx
## 处理数据 入口
**getDerivedStateFromProps**

## Tree Node 处理逻辑
### 判断是否需要更新数据
#### **needSync**
```Javascript
function needSync(name: string) {
    // 1. props前面没值，并且 props里面有值（第一次）  
    // 2. 数据 直接比较 相同否 第一次是 fieldNames 有可能变化的情况 treeData 变化的情况
    //    children的提示
      return (!prevProps && name in props) || (prevProps && prevProps[name] !== props[name]);
    }
```

#### **convertDataToEntities**

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aae2911141d84948a19382ce49fd4edc~tplv-k3u1fbpfcp-watermark.image)
转化为

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f77f1c9a77b546ec9392f1a5987b15f6~tplv-k3u1fbpfcp-watermark.image)
#### 函数解析
```Typescript
/**
 * Convert `treeData` into entity records.
 * 生成一个 key entries 的一个映射
 */
export function convertDataToEntities(
  dataNodes: DataNode[],
  {
    initWrapper,
    processEntity,
    onProcessFinished,
    externalGetKey,
    childrenPropName,
    fieldNames,
  }: {
    initWrapper?: (wrapper: Wrapper) => Wrapper;
    processEntity?: (entity: DataEntity, wrapper: Wrapper) => void;
    onProcessFinished?: (wrapper: Wrapper) => void;
    externalGetKey?: ExternalGetKey;
    childrenPropName?: string;
    fieldNames?: FieldNames;
  } = {},
  /** @deprecated Use `config.externalGetKey` instead */
  legacyExternalGetKey?: ExternalGetKey,
) {
  // Init config
  const mergedExternalGetKey = externalGetKey || legacyExternalGetKey;

  const posEntities = {};
  const keyEntities = {};
  let wrapper = {
    posEntities,
    keyEntities,
  };

  if (initWrapper) {
    wrapper = initWrapper(wrapper) || wrapper;
  }

  traverseDataNodes(
    dataNodes,
    item => {
      const { node, index, pos, key, parentPos, level } = item;
      const entity: DataEntity = { node, index, key, pos, level };

      const mergedKey = getKey(key, pos);
       
       // Pos 的key Map 映射
      posEntities[pos] = entity;
       // key 的key Map 映射
      keyEntities[mergedKey] = entity;

      // Fill children
      entity.parent = posEntities[parentPos];
      if (entity.parent) {
        entity.parent.children = entity.parent.children || [];
        entity.parent.children.push(entity);
      }

      if (processEntity) {
        processEntity(entity, wrapper);
      }
    },
    { externalGetKey: mergedExternalGetKey, childrenPropName, fieldNames },
  );

  if (onProcessFinished) {
    onProcessFinished(wrapper);
  }

  return wrapper;
}
```

### traverseDataNodes => processNode 递归处理节点
其中各种key 的复写或者使用用户的key 的这些操作可以忽略
只关注逻辑处理部分


## 处理默认展开的 expandedKeys
主要看 这个方法 **conductExpandParent** 
传入两个数据
1. keyList : 对应默认展开的key 数组
2. keyEntities： 由之前 convertDataToEntities 生成的key 对应数据的映射。这里之前这么做的原因在于树形数据的查询会变成O(1) 的复杂度
```Typescript

/**
 * If user use `autoExpandParent` we should get the list of parent node
 * @param keyList
 * @param keyEntities
 */
export function conductExpandParent(keyList: Key[], keyEntities: Record<Key, DataEntity>): Key[] {
  const expandedKeys = new Set<Key>();

  function conductUp(key: Key) {
    if (expandedKeys.has(key)) return;

    const entity = keyEntities[key];
    if (!entity) return;

    expandedKeys.add(key);

    const { parent, node } = entity;

    if (node.disabled) return;

    if (parent) {
      conductUp(parent.key);
    }
  }

  (keyList || []).forEach((key) => {
    conductUp(key);
  });

  return [...expandedKeys];
}
```

## 展开节点，扁平化数据 flattenNodes
###  **flattenTreeData**

递归调用 **dig 函数**

把数据从

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/00f25e5c16524c1a9d6cc91687e606a2~tplv-k3u1fbpfcp-zoom-1.image)

转成

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a00de5febca40c6981ed1dc1f196c70~tplv-k3u1fbpfcp-zoom-1.image)

### dig 函数解析

#### **getPosition** 函数用来 给节点添加 pos 用于表示节点的具体位置 以 - 作为连接符
两个参数，parent 如果有的话。作为pos 的前半部分，index 作为表示节点在数组中的位置
###### 目的：
1. 标识层级
2. 标识位置。后续提供给查找节点等提供便利

####  **getKey** 函数作用
1. 获取key ，如果节点没有key 默认以上面**getPosition** 函数结果作为Key

#### flattenList
作为扁平化后的数据存储。每次遍历都会往 flattenList push 一个元素

#### FlattenNode
``` Typescript
const flattenNode: FlattenNode = {
        ...omit(treeNode, [fieldTitle, fieldKey, fieldChildren] as any),
        title: treeNode[fieldTitle],// 节点显示的label
        key: mergedKey, // getKey 结果
        parent, // 父级节点
        pos, // getPosition 结果
        children: null,
        data: treeNode, // 原始数据
        // 标识是否为数组的起始位置
        isStart: [...(parent ? parent.isStart : []), index === 0],
        // 标识是否为当前数组的结尾
        isEnd: [...(parent ? parent.isEnd : []), index === list.length - 1], 
      };
```

#### 判断 **expandedKeySet**
这个值有两种情况，一种是 boolean的 true|false 一种是 数组形式，函数上面 new Set是为了去重
如果 **expandedKeySet** 有值，便继续往下递归
// 为什么 expandedKeySet 没有值。不继续往下递归。稍等看下


其他的key处理相对类似。主要是基于前面处理树状结构数据和节点数据生成的 keymap 和 list等。以下就不做解析了，具体主要代码贴在这里
## selectedKeys 已选key
主要处理方法是 calcSelectedKeys

```Typescript
/**
 * Return selectedKeys according with multiple prop
 * @param selectedKeys
 * @param props
 * @returns [string]
 */
export function calcSelectedKeys(selectedKeys: Key[], props: TreeProps) {
  if (!selectedKeys) return undefined;

  const { multiple } = props;
  if (multiple) {
    return selectedKeys.slice();
  }

  if (selectedKeys.length) {
    return [selectedKeys[0]];
  }
  return selectedKeys;
}
```

## checkedKeys 处理 主要处理函数
```Typescript
/**
 * Parse `checkedKeys` to { checkedKeys, halfCheckedKeys } style
 */
export function parseCheckedKeys(keys: Key[] | { checked: Key[]; halfChecked: Key[] }) {
  if (!keys) {
    return null;
  }

  // Convert keys to object format
  let keyProps;
  if (Array.isArray(keys)) {
    // [Legacy] Follow the api doc
    keyProps = {
      checkedKeys: keys,
      halfCheckedKeys: undefined,
    };
  } else if (typeof keys === 'object') {
    keyProps = {
      checkedKeys: keys.checked || undefined,
      halfCheckedKeys: keys.halfChecked || undefined,
    };
  } else {
    warning(false, '`checkedKeys` is not an array or an object');
    return null;
  }

  return keyProps;
}
```


至此，rc-tree 的数据整理部分就分析到这里
主要要理解的事情是，如何把数据的处理的复杂度降低到最小。理解前面的节点处理 convertDataToEntities 函数对后面函数的作用就理解了大半.






