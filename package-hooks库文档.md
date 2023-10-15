# @ifca/hooks

## usePopupListRequestHandle

### 1、代码

```ts
/**
 * 一维数组转树形数据
 * @example
 * // 示例1：
 * let list = [ { id: '1-1', pid: null }, { id: '2-1', pid: '1-1' }, { id: '2-21', pid: '1-1' }, { id: '3-1', pid: '2-1' } ]
 * listToTree(list)
 * // 输出： [{"id":"1-1","pid":null,"children":[{"id":"2-1","pid":"1-1","children":[{"id":"3-1","pid":"2-1","children":[]}]},{"id":"2-21","pid":"1-1","children":[]}]}]
 * @param list
 * @param options
 */
export const listToTree: ListToTree<any> = <
  T extends { [key: string]: string }
>(
  list: T[],
  options?: {
    id?: string;
    pid?: string;
    children?: string;
  }
) => {
  const map: { [key: number | string]: TreeNode<T> } = {};
  const roots: TreeNode<T>[] = [];
  const idKey = options?.id || 'id';
  const pidKey = options?.pid || 'parentId';
  const childrenKey = options?.children || 'children';

  // 创建节点映射
  list.forEach((item) => {
    const id = item[idKey];
    map[id] = { ...item, [childrenKey]: [] };
  });

  // 构建树
  list.forEach((item) => {
    const id = item[idKey];
    const pid = item[pidKey];
    const node = map[id];

    if (pid !== null) {
      const parent = map[pid];

      if (parent) {
        parent[childrenKey].push(node);
      }
    } else {
      roots.push(node); // 根节点
    }
  });

  return roots;
};
```

```ts
/**
 * 用于ProFormSelect和ProFormTreeSelect的请求方法：
 * 接口返回的结构大部分是：{
 *    data: [],
 *    groupCount: -1,
 *    summary: ""
 *    totalCount: 0,
 * }
 */
type PopupResult = Promise<any[]>;
type PopupParams = Record<string, any>;
type PopupFunction = (params: Params) => PopupResult;
type PopupExtraParams = {
  dealData?: (val: any[]) => any[]; // 如果返回的数据需要处理，在外部处理完再返回
  type?: 'treeSelect' | 'tree';
  id?: string; // 根据哪个字段与parentId关联父子关系
  pid?: string; // 接口返回的parentId
  fieldNames?: {
    label?: string;
    value?: string;
  }; // 组件显示的label与value字段
};
type UsePopupListRequestHandle = (
  service: any,
  extraParams?: PopupExtraParams
) => [PopupFunction];

export const usePopupListRequestHandle: UsePopupListRequestHandle = (
  service: any,
  extraParams?: PopupExtraParams
) => {
  const {
    dealData,
    type = 'tree',
    id = 'id',
    pid = 'parentId',
    fieldNames,
  } = extraParams ?? {};

  const popupRequest = useCallback(
    async (params: PopupParams): PopupResult => {
      const res =
        (await service({
          ...params,
        })) ?? {};

      const finalRes = (res.data ?? []).map((item: any) => {
        return {
          ...item,
          label: fieldNames?.label ? item[fieldNames?.label] : item['name'],
          value: fieldNames?.value ? item[fieldNames?.value] : item['id'],
        };
      });

      if (dealData) {
        if (type === 'treeSelect') {
          return listToTree(dealData(finalRes), {
            id,
            pid,
          });
        } else {
          return dealData(finalRes);
        }
      }

      if (type === 'treeSelect') {
        return listToTree(finalRes, {
          id,
          pid,
        });
      }

      return finalRes;
    },
    [service]
  );

  return [popupRequest];
};
```

### 2. 参数解释

| 参数       | 说明                                                                                                  | 默认值                        | 类型                             | 是否必传 |
| ---------- | ----------------------------------------------------------------------------------------------------- | ----------------------------- | -------------------------------- | -------- |
| type       | 树选择或选择器                                                                                        | 'tree'                        | 'treeSelect' or 'tree'           | 否       |
| pid        | 只对'treeSlect'类型有效，当需要把接口的一维数组转换为树形结构时使用，根据接口返回实际传递该字段       | 'parentId'                    | string                           | 否       |
| id         | 只对'treeSlect'类型有效，当需要把接口的一维数组转换为树形结构时使用，根据哪个字段与父 id 关联父子关系 | 'id'                          | string                           | 否       |
| dealData   | 把请求数据回调出去，在外部对通过 request 获取的数据进行处理，比如过滤等                               | -                             | (val: any[]) => any[]            | 否       |
| fieldNames | 组件展示的 label 与 value 字段，根据接口返回传递                                                      | { label: 'name', value: 'id'} | {label?: string;value?: string;} | 否       |

### 3. 例子:

#### （1）Form 表单使用：

```tsx
import { usePopupListRequestHandle } from '@ifca/hooks';
const Demo = () => {
  const [ProfileRequest] = usePopupListRequestHandle(getAppProductProfile, {
    fieldNames: {
      value: 'code',
    },
    id: 'code',
    pid: 'parentCode',
    type: 'treeSelect',
    dealData: (res: any) => res.filter((item: any) => item.type === 2),
  });

  return (
    <ProFormTreeSelect
      name="appCode"
      label="应用编码"
      request={ProfileRequest}
      fieldProps={{
        showSearch: true,
        treeNodeFilterProp: 'name',
      }}
    />
  );
};
```

#### （2）Table 的 Columns 使用：

```tsx
import { usePopupListRequestHandle } from '@ifca/hooks';
const [ProfileRequest] = usePopupListRequestHandle(getAppProductProfile, {
  fieldNames: {
    value: 'code',
  },
  id: 'code',
  pid: 'parentCode',
  type: 'treeSelect',
  dealData: (res: any[]) => res.filter((item: any) => item.type === 2),
});

const columns = [
  {
    title: '应用编码',
    dataIndex: 'appCode',
    valueType: 'treeSelect',
    request: ProfileRequest,
    fieldProps: {
      showSearch: true,
      multiple: true,
      treeNodeFilterProp: 'name',
    },
    request: ProfileRequest,
  },
];
```
