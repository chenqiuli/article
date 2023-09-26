## antd Procomponent 使用总结记录

#### antd Procomponent 是基于 Ant Design 封装的高阶组件，开箱即用。本文只是对在工作中使用 antdPro 的小小记录，若需使用 antd Procomponent，请移步[官网](https://procomponents.ant.design/docs)

### ProTable

Antd Table 的高阶组件，添加可配置化搜索区域，自带 request 请求实现表格的搜索，在使用的过程中发现了在 antd 的 Table 升级的地方：

1. 在新增和查看表单的时候，两个表单页面是共用的，但通常在查看表单页面表格的列会与新增的不同，这就涉及到了表格列的动态显隐，ProTable 提供了一个很好的参数：`hideInTable`，用起来太爽了吧，少了很少 if 条件去动态判断 columns。

```tsx
const columns = [
  {
    title: '必须拍照',
    dataIndex: 'isPZ',
    key: 'isPZ',
    hideInTable: check ? true : false,
    render: (_, record: InspectTableType) => {
      if (!record.children) {
        return <span>是</span>;
      }
      return null;
    },
  },
];
```

2. 在以前使用 antd Table 的时候，如果遇到某一列的数据是下拉框返回的数据需要自己去做映射的时候，都需要在 render 里面处理返回，现在 ProTable 提供了一个参数：`valueEnum`，可以直接映射到枚举类型。

```tsx
const statusList = {
  OPEN: {
    text: '已启用',
    status: 'OPEN',
  },
  STOP: {
    text: '已停用',
    status: 'STOP',
  },
};
const columns = [
  {
    title: '状态',
    dataIndex: 'status',
    valueType: 'select',
    valueEnum: statusList,
  },
];
```

3. ProTable 最强大的地方在于集合了搜索区的搜索条件、表格列的过滤条件和排序规则，让我们只需要注意业务本身而不是 CRUD，这个参数就是`request`，`request` 方法返回一个 Promise 对象，在查询表格和参数改变的时候重新执行，项目里对`request`方法进行了封装：

```ts
const request = async (
  service: any,
  params: Params,
  sort: Sort,
  filter: Filter,
  filterOptions?: FilterOptions
): Promise<any> => {
  let filterArray: any[] = [];
  let sortArray: any[] = getSortArray(sort);
  filterArray = [
    ...getParamsArray(params, filterOptions),
    ...getFilterArray(filter),
  ];

  return await service({
    RequireTotalCount: true,
    Skip: ((params.current || 1) - 1) * (params.pageSize || 0),
    Take: params.pageSize,
    Filter: JSON.stringify(filterArray),
    Sort: JSON.stringify(sortArray),
  });
};

export const usePagingRequestHandle: UsePagingRequestHandle = (
  service: any,
  filterOptions
) => {
  const pagingRequest = useCallback(
    async (params: Params, sort: Sort, filter: Filter): Result => {
      const res = await request(service, params, sort, filter, filterOptions);

      return {
        data: res.data,
        success: true,
        total: params.pageSize ? res.totalCount : null,
      };
    },
    [service]
  );

  return [pagingRequest];
};
```

页面中引入即可使用：

```tsx
const [pagingRequest] = usePagingRequestHandle(getAppUserData);

<ProTable<ColumnsType> request={pagingRequest} />;
```

### ProForm/ModalForm/DrawerForm

大量的弹窗页面嵌套表单，ModalForm 和 DrawerForm 整合了 ProForm 和 Modal；ProForm 的页脚按钮自动在左边，若想挪到右边，需要用 `submitter` 的 `render` 重写，而 ModalForm 和 DrawerForm 页脚默认在右边。

1. 修改页脚的按钮内容：

```tsx
<ProForm
  submitter={{
    // 配置按钮文本
    searchConfig: {
      resetText: '重置',
      submitText: '提交',
    },
    // 配置按钮的属性
    resetButtonProps: {
      style: {
        // 隐藏重置按钮
        display: 'none',
      },
    },
    submitButtonProps: {},
    // 完全自定义整个区域
    render: (props, doms) => {
      return [
        <button
          type="button"
          key="rest"
          onClick={() => props.form?.resetFields()}
        >
          重置
        </button>,
        <button
          type="button"
          key="submit"
          onClick={() => props.form?.submit?.()}
        >
          提交
        </button>,
      ];
    },
  }}
/>
```

### ProFormTimePicker

常见的需求：对结束时间作限制，不能早于当前时间，使用 `disableTime`

```tsx
const disabledTime = (startTime: string | undefined) => {
  return {
    disabledHours: () => {
      let hours = [];
      let timeArr = startTime?.split(':') ?? [];
      for (let i = 0; i < parseInt(timeArr[0]); i++) {
        hours.push(i);
      }
      return hours;
    },
    disabledMinutes: (selectedHour: number) => {
      let timeArr = startTime?.split(':') ?? [];
      let minutes = [];
      if (selectedHour === parseInt(timeArr[0])) {
        for (let i = 0; i < parseInt(timeArr[1]); i++) {
          minutes.push(i);
        }
      }
      return minutes;
    },
  };
};

<ProFormTimePicker
  fieldProps={{
    format: 'HH:mm',
    value: record.endTime
      ? (moment(record.endTime, 'HH:mm') as any)
      : undefined,
    disabledTime: () => disabledTime(record.startTime),
  }}
  disabled={typeof record.startTime !== 'string'}
  style={{ width: 100 }}
/>;
```

### ProForm.Item 与 Tooltip 结合

项目中设计团队想使用 tooltip 去展示 ProForm.Item 中的校验错误信息，如果使用 ProForm 自带的 rules 里面的 validator 校验，错误信息的展示方式是红色爆红在控件下方的，所以需要去 DrawerForm 标签上使用 onValuesChange 监听这个字段。校验的规则是必须满足大写小写数字特殊字符长度 6 位，因此自己写一个，记录一下，方便以后用到哈哈：

```tsx
const pwdRegs = [
  {
    reg: bigCharReg,
    error: '密码未包含大写字母',
  },
  {
    reg: smallCharReg,
    error: '密码未包含小写字母',
  },
  {
    reg: digitReg,
    error: '密码未包含数字',
  },
  {
    reg: specialCharReg,
    error: '密码未包含特殊字符',
  },
  {
    reg: morethanSixReg,
    error: '密码未达到6位',
  },
];
const { run: validatepwd } = useDebounceFn(
  (value: string) => {
    if (value) {
      settooltipOpen(true);
      const errorItem = pwdRegs.find((item) => !item.reg.test(value));
      if (errorItem) {
        setpwdError(errorItem.error);
        setpwdStatus(false);
        return false;
      } else {
        settooltipOpen(false);
        setpwdStatus(true);
        setpwdError('该密码合格');
        return true;
      }
    }
    settooltipOpen(false);
    setpwdError('');
    return true;
  },
  {
    wait: 500,
  }
);

<DrawerForm
  form={form}
  onValuesChange={(changeValues) => {
    if (changeValues.password) {
      validatepwd(changeValues.password);
    }
  }}
>
  <Tooltip
    title={
      pwdStatus ? (
        <>
          <CheckCircleOutlined style={{ color: '#52c41a' }} />
          <span style={{ marginLeft: 4 }}>该密码合格</span>
        </>
      ) : (
        <>
          {pwdError.length > 0 && (
            <CloseCircleOutlined style={{ color: '#f5222d' }} />
          )}
          <span style={{ marginLeft: 4 }}>{pwdError}</span>
        </>
      )
    }
    placement="bottom"
    trigger="hover"
    color="#464c59"
    open={tooltipOpen}
  >
    <ProForm.Item
      name="password"
      label={`登录密码`}
      rules={[
        {
          required: true,
          message: '请输入',
        },
      ]}
    >
      <Input.Password
        autoComplete="new-password"
        placeholder="大小写字母、数字和特殊字符(!@#$%^&*/.?)，长度为6位"
      />
    </ProForm.Item>
  </Tooltip>
</DrawerForm>;
```

### ProFormText.Password 自动填充浏览器账号密码

被这个问题困扰了好久，原因竟然是在 Input.Password 所在的地方上面会自动填充账号，Input.Password 填充密码，查找了 MDN，一定要在 Input.Password 上添加 `autoComplete="new-password"`，`autoComplete="off"` 亲测无效。

```tsx
<ProFormText.Password
  fieldProps={{
    autoComplete: 'new-password',
  }}
  name="password"
  label="密码"
/>
```
